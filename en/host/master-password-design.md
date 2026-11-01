# Optional Master Password: Design and Implementation

> Date written: 2026-10-25 (draft); rewritten against the implementation on 2026-10-31　　Baseline code: host `feat/master-password` (based on `dev` 0ffdecf3)
>
> Status: **implemented (second step)**. The maintainers settled the six decision points of §9 on 2026-10-31 (along the draft's leanings); wherever the text and the implementation differed, the text below follows the implementation.
>
> Related: [System-keychain and sudo credential fill feasibility research](system-keychain-and-sudo-credential-fill-feasibility-research.md) (its 2026-10-25 erratum at the top describes how the first step was implemented),
> [Credential manager integration design](credential-manager-integration-design.md) §5.5 / §5.7 (build points, interactive / non-interactive resolution), [settings-audit](settings-audit.md) R-04,
> [Interaction and UI specs](interaction-and-ui-specs.md) §7 / §14.7 (the status-bar lock entry, the settings rows and the dialogs).

## 0. Conclusion first

- The master password is an **optional** second layer, off by default. Once on, what the system keychain (macOS keychain / Secret Service) or the Windows DPAPI file holds
  is no longer the local encryption key itself but **the local key wrapped with a key derived from the master password**: both the keychain entry and the master password are needed to open it.
- **Existing `enc1:` ciphertexts do not change by a single byte.** The master password wraps the local key (the data encryption key, DEK), not each ciphertext — turning the master
  password on, off or changing it rewrites only a few dozen bytes of wrapping, takes seconds and never touches the database. KeePass / Bitwarden / 1Password do the same.
- Unlocking: an unlock dialog appears once after the main window is up (cancellable); after cancelling, or while locked, the app keeps working — saved passwords just cannot be read — and a lock entry stays on the left of the status bar.
  Automatic reconnects, the batch reconnect after waking from sleep, tunnel auto-rebuild and automatic Gist sync treat it as "locked" and **never show the dialog**; plugins reading or writing secrets **wait for an interactive unlock** in this round (§6).
- **Forgetting the master password = the saved secrets are void; there is no back door.** Turning it on creates a recovery code by default (skippable), which opens it too; with neither, "Forgot the master password?" starts a new key,
  keeping the old wrapping, header file and the old ciphertexts in the database (they can still be opened if the password comes back to mind).

## 1. Where the first step left things

- `MasterKeyResolver` resolves a **key ring** (one or more 32-byte DEKs; the first encrypts, decryption tries each in turn); on macOS / Linux each key is one keychain entry
  (id = the first 128 bits of a domain-separated SHA-256(DEK)), and `secret.key.ref` in the data directory lists only the ids; on Windows it is the DPAPI-wrapped `secret.key`; when the keychain is unavailable it falls back to a plaintext `secret.key`.
- `ISecretProtector` is a synchronous `string → string`, with callers all over the session repository, shared credentials, Gist, plugin secrets and the plugin trust store; `TryUnprotect` can say truthfully that something could not be decrypted.

## 2. Threat model: what a master password adds

| Scenario | First step (keychain) | With a master password |
| --- | --- | --- |
| The data directory is copied / backed up / synced to a cloud drive | Stopped | Stopped |
| Another program running as the same user reads the keychain entry (macOS entries are open to any application; Secret Service is open to processes in the session once unlocked; DPAPI to the user's own processes) | **Not stopped** (same level as the former 0600 file) | Stopped (the master password is needed too) |
| Keychain / keyring files leak together with the login password; DPAPI after a domain administrator resets the account | Not stopped | Stopped |
| Someone sits down at your logged-in, unlocked computer and opens VelaShell | Not stopped | Stopped while locked (§5 automatic locking) |
| Reading process memory or keylogging after unlocking | Not stopped | **Not stopped** (stated plainly, not promised) |
| In-process plugins (same process as the host) | Not stopped | Not stopped after unlocking; isolated plugins still reach only their own secrets through `ISecretsApi` |

## 3. Key hierarchy and storage format

```
master password ──Argon2id(salt, m, t, p)──► KEK (32 bytes, zeroed as soon as wrapping / unwrapping is done)
recovery code (128 random bits) ──HKDF-SHA256(recovery-salt)──► RK (likewise)
                                              │ AES-256-GCM (random 12-byte nonce, associated data = VSMP1 ‖ kind ‖ wrap id)
DEK (every key in the first step's ring) ──────►  record = kind ‖ wrap id ‖ nonce ‖ tag ‖ GCM(KEK or RK, DEK)
```

- The **KDF header** (salt and parameters, not secret) lives in `secret.key.mp` in the data directory: `version=1`, `kdf=argon2id`, `m`, `t`, `p`, `salt` (16 random bytes, base64),
  `recovery-salt` (only when a recovery code was created) and `state` (`active` / `enabling` / `disabling`, see §7). Parameters out of a sane range (m above 1 GiB, t above 64, p above 16) are treated as unreadable —
  so a header edited to huge values cannot stall unlocking. The whole key ring shares one KEK (one derivation opens them all).
- The **wrapping** (the value of each keychain entry, or the content of the key file) changes from "32-byte DEK" to the magic `VSMP1` ‖ one or more **records** of 69 bytes each:
  kind (1 byte, `P` = master password, `R` = recovery code) ‖ wrap id (8 bytes, = the first 64 bits of a domain-separated SHA-256(kind ‖ the matching salt)) ‖ nonce (12) ‖ tag (16) ‖ ciphertext (32).
  Normally two (one for the master password, one for the recovery code; one if the code was skipped). While changing the master password there is briefly one more (§7); records whose wrap id does not match the current header are leftovers of that transition, unused when unlocking and removed after a successful unlock.
- The entry id is still derived from the DEK and does not change — so the first step's rule "an entry whose content does not match its id is neither used nor touched" protects wrapped entries for free: an older version that does not know `VSMP1` only reports "not found" and creates a new key, never overwriting.
  After unwrapping the key proves itself against the id the same way: a key that unwraps but is not the one the entry is named after (the entry was swapped) is not used.
- **Windows**: the content of `secret.key` changes from `DPAPI(DEK)` to `DPAPI(VSMP1 ‖ records)`; the same goes for platforms that fell back to a plaintext file (`VSMP1 ‖ records`, where the master password is then the only barrier). Replacing the key file always writes a temporary file, flushes it and renames it over.
- Verification: the GCM authentication tag is the master password check; no separate verifier is stored (one less thing to compare against offline). The recovery code is 128 random bits (plus one checksum byte, written as 28 Crockford base32 characters in groups of 4)
  and does not fear offline enumeration, so it is derived with HKDF rather than Argon2id; the checksum byte only lets a one-character typo be reported as "mistyped" instead of "wrong", and does not count as a failed try.
- **The source of truth is the wrapping format itself**, not `AppSettings.MasterPasswordProtection`: settings import and Gist sync can both change that boolean.
  The field is only a display mirror (the settings page writes back the actual state), registered in `SettingsCategories.StateFields` (not exported, import keeps the local value, not audited); cloud sync clears it before pushing and keeps the local one when pulling.
- **How "on" is decided** (at resolution): a wrapping was found → on; the header exists and no wrapping was found but some entry could not be asked this time (keychain unavailable) → treated as on, locked, unable to tell whether it can be unlocked (`KeysUnavailable`);
  the header exists, no wrapping, and every slot was asked → a header in `enabling` / `disabling` is deleted (enabling broke off before wrapping anything / disabling already wrote everything back), an `active` one is renamed to `secret.key.mp.orphaned-<timestamp>` and kept, the app continues as off, and the message center explains it;
  wrappings without a header (`HeaderMissing`) or with an unreadable header (`HeaderUnreadable`, kept as is) → on and impossible to unlock; only "Forgot the master password?" helps.
- While on, resolution **neither creates nor migrates any key**: a new key the master password does not cover would quietly switch the protection off. The first step creates a temporary local key when the keychain is unavailable; that is not allowed while on.
- Alternative (not adopted): with the master password on, keep the key only as a wrapped file in the data directory and delete the keychain entries — the data directory plus the master password would move to another machine,
  but with a weak master password this falls back to "get the directory and brute-force it offline", the opposite of what the first step is for; moving machines already has connection export and end-to-end sync.

## 4. KDF: Argon2id

- Parameters: m = 64 MiB, t = 3, p = 1 (the second recommendation of RFC 9106 §4 is m = 64 MiB, t = 3, p = 4; the OWASP minimum is m = 19 MiB, t = 2, p = 1), about 0.3–0.6 s on a mainstream laptop.
  The maintainers decided on 2026-10-31 not to lower them for low-end machines. Parameters are stored in the header; when the header's parameters are not the current defaults, the next successful unlock with the master password re-wraps with new parameters and a new salt (rehash, using the three steps of "Change" in §7), with no one-off migration.
- Implementation: the BCL has no Argon2. The host already carries BouncyCastle.Cryptography through `VelaShell.Ssh` (it provides Argon2id for `.ppk` v3), so Infrastructure references the same package explicitly and adds no dependency;
  the comment in `src/Directory.Packages.props` records this new use (the "no second use" in `VelaShell.Ssh/README.md` refers to the inside of the SSH library).
- Derivation runs on the thread pool (64 MiB of memory, a few hundred ms of CPU), with the dialog in a busy state and its buttons disabled; once started a derivation runs to the end, even if the wait is cancelled.
- Online guessing: after the n-th consecutive wrong entry the next try waits 2^(n-1) seconds (1, 2, 4… capped at 30 s), counted down in the dialog; nothing is locked out or deleted, and the count lives only in this process.
  Offline guessing is held back only by the parameters and password strength: a new master password shorter than 12 characters shows "at least 12 characters or a passphrase" — a hint, not a block. The password box is `SecurePasswordBox` and takes printable English characters only (same as the other password boxes).

## 5. Unlocking, the locked state and saving

- **When to unlock** (decided 2026-10-31): one unlock dialog after the main window appears, cancellable; after cancelling, a lock entry stays on the left of the status bar (click it to unlock while locked, or to lock right away while unlocked).
  Sessions restored at start-up wait for the unlock at the same time and share that one dialog.
- **Semantics while locked**: `AesSecretProtector` has no usable key ring → `TryUnprotect` is false and `Unprotect` returns the ciphertext as is (exactly the first step's "key not obtained" path, which callers already handle);
  `Protect` throws `SecretsLockedException` for new plaintext (a value that is already ciphertext is still returned as is, idempotently); `ISecretProtector.IsLocked` says so.
- **Repositories freeze secret fields at the stored copy while locked**: a connection's password / key passphrase / plugin secrets, a shared credential's password / passphrase, the proxy password in settings — a value that is already ciphertext is stored as is,
  anything else takes the copy currently in the database (empty if none), and the other fields are saved as usual. New plaintext cannot get in, and the old value is not replaced by an empty copy (one with "Remember password" off, one stripped after a cancelled unlock)
  or by plaintext decrypted before locking that is still in memory. The one-off import of an old `sessions.json` is postponed until unlocked.
- **Asynchronous gate** `IMasterPasswordGate { bool IsEnabled; bool IsLocked; Task<bool> EnsureUnlockedAsync(bool interactive, CancellationToken); event Unlocked; event Locked; }`,
  with the management entry `IMasterPasswordService` (unlock, unlock with the recovery code, lock now, turn on / off / change / forget, status). Concurrent interactive requests share the result of one dialog; after the user cancels, all waiting requests get false together instead of one dialog each;
  the UI layer registers `IMasterPasswordPrompt` (modal to the currently active window); when the container has none (headless, tests) interactive requests can only be false.
- **Entry points that use secrets go through the gate**: the connection build points (connections you start, opening SFTP / FTP / plugin documents and plugin terminals, background connections of the tunnel panel) each get one line in front of the existing build point:
  when the connection would use saved secrets (a stored password / passphrase / plugin secrets, a shared-credential reference, a jump host) and it is locked, the dialog comes first; once unlocked, ciphertext loaded while locked is replaced with the plaintext the database can now decrypt;
  if cancelled, **plaintext left in memory from before locking is not used either** — this copy's saved secrets are stripped, the sign-in dialog asks for this one time, and saving it back after connecting leaves the stored copy alone.
  Editing a connection that has saved secrets, saving a shared credential, importing connections (Xshell / WinSCP / OpenSSH / connection files), saving the Gist token / passphrase and a manual sync also show the dialog first; cancelling saves the other fields and says plainly "Not unlocked: the password was not saved this time".
- **Automatic locking** (decided 2026-10-31: off by default, optional): "Lock after idle (minutes)" looks only at keyboard, mouse, wheel and touch input in this app's windows (tunnel class handlers on every top-level window);
  terminal output streaming by does not count; "Lock when the system locks" can currently only be read on Windows (`SessionFlags` of `WTSQuerySessionInformation`; if it cannot be read nothing is locked);
  macOS / Linux have no reader yet, so the settings page does not show that option (so it cannot be on without working). The check runs every 5 seconds on a thread-pool timer; unlocking resets the idle timer.
- **Locking = zeroing the DEKs in memory** (`CryptographicOperations.ZeroMemory`) and going back to locked; encryption and decryption run under the same lock, so locking never interleaves with an encryption in progress.
  The session tree is then re-read (repositories return ciphertext while locked), and manual reconnects and new connections need an unlock. **Not cleared**: connected sessions and open FTP connection pools keep working;
  the proxy password stays in memory (otherwise every proxied connection would break); the in-memory copy of an open tab's profile is not scrubbed, but reconnecting or cloning it while locked goes through the gate.
- **Unlock dialog**: modal to the currently active window, `SecurePasswordBox`, never copied to the clipboard or history; can switch to "Unlock with the recovery code"; when the header is missing / unreadable or the keychain did not open it says so directly, leaving only Cancel or "Forgot the master password?".

## 6. Automatic reconnects and background tasks

The same table as the [credential manager integration design](credential-manager-integration-design.md) §5.7 (its "vault locked" is "master password locked" here):

| Trigger | Mode | While locked |
| --- | --- | --- |
| A connection started by the user, a manual reconnect (Ctrl+R / Enter / "Reconnect"), opening SFTP / FTP / plugin documents, plugin `Sessions.OpenAsync`, restoring sessions at start-up, starting a tunnel manually | Interactive | Show the unlock dialog (one at a time) |
| The automatic reconnect timer, `ReconnectAllAfterResume` after waking from sleep | Non-interactive | No dialog; the tab's failure overlay says "Locked: automatic reconnects never show the unlock dialog. Once you unlock this tab is retried once; you can also reconnect with Enter or Ctrl+R"; after a successful unlock each pending tab is retried once |
| Tunnel `AutoReconnectAsync` | Non-interactive | No dialog, no back-off; the panel says it rebuilds after unlocking; each is retried once after unlocking |
| Automatic Gist sync | Non-interactive | Skip this round and log a line (`[Sync] Skipped: saved secrets are locked…`); a manual "Sync now" shows the dialog first |
| Plugins reading / writing secrets (`ISecretsApi.GetAsync` / `SetAsync`) | Interactive (decided 2026-10-31) | Wait for an interactive unlock first (show the dialog, or wait for the one already open); cancelling throws `PluginPermissionDeniedException` (the closest meaning the current SDK can express — "the user did not allow the read", wire code `permission-denied`, restored the same way for isolated plugins over RPC). **Never return null**: the plugin would think the secret does not exist, ask the user to enter it again and overwrite the old value. Deleting needs no decryption and goes ahead |
| Connection diagnostics | — | Not gated in this round: while locked, step 4 gets ciphertext it cannot decrypt and is marked as needing the user, as "the saved password cannot be decrypted" (same as the first step) |
| Plugin trust store | — | Start-up does not wait: discovery and bundled plugins proceed; plugins installed in the user directory have their receipts verified only after unlocking (no dialog). Plugin-manager queries do not wait and say "locked" rather than "unreadable" (that message advises a reset); installing a plugin and trusting a publisher show the dialog first |

Same reasoning as the credential manager design: when the network comes back and 20 tabs reconnect at once, an unlock dialog the user cannot attribute to anything must not pop up (phishing concerns).

The SDK also has a separate "locked" contract written (velashell-plugin-sdk branch `feat/secrets-locked`, available from TBD): `SecretsLockedException` (derives from `InvalidOperationException`),
`ISecretsApi.SupportsLockedState` (default false), `HandshakeResponse.SupportsSecretsLocked`, the wire code `secrets-locked` and `FakeSecrets.IsLocked`. A host that declares the capability throws it from reads / writes while locked instead of prompting;
**the host does not use it in this round**; whether background secret reads become non-interactive is decided after the SDK release.

## 7. Turning on / off / changing / forgetting

Every step touches only the wrapping, never the ciphertext; each slot counts only after "write the new value and read it back byte for byte"; if any slot cannot be asked right now (system keychain unavailable), nothing is done —
that slot's state is unknown, and changing the others could leave a key that can never be opened again.

- **Turn on** (Settings → General → Privacy & Security, restoring the R-04 row): enter twice, strength hint, tick "I understand: if I forget the master password (and have no recovery code), saved passwords cannot be recovered", "Also create a recovery code" ticked by default →
  read every slot afresh (every entry in the list + the local key file; every key in use must be in some slot, otherwise refuse) → write a header with `state=enabling` first → derive the KEK (and RK) →
  write each slot's wrapping and check it → switch the header to `active`. If a slot cannot be written: slots already wrapped are written back as plaintext and the header deleted, back to "off" as a whole; if the rollback cannot be written either, it says so plainly — "the master password is now on; unlock with the one you just entered"
  (the header stays `enabling`, and the next unlock wraps the slots still in plaintext). The recovery code is shown only in this step.
- **Turn off**: enter the current master password → open every wrapping (each must open) → mark the header `disabling` → write each plaintext DEK back (the first step's form) → delete the header. A failure halfway writes the original wrapping back and sets the header back to `active`;
  a power cut after everything was written back makes the next start (no wrapping left, header `disabling`) delete the header, and one halfway through is finished at the next unlock with the master password.
- **Change**: old + new → open every wrapping → new header (new salt, default parameters, the recovery code's salt kept) → ① **add** the new master password's record to each slot (the old record and the recovery record stay) → ② swap the header atomically → ③ remove the old record.
  Broken off after ① the old master password still opens it (the new records do not match the current header and are cleaned at the next unlock); after ② the new one opens it; an unfinished ③ only leaves an unused record. Failures in ① and ② write the original bytes back and leave the header alone. The recovery code is unaffected.
- **After unlocking with the recovery code**: you are asked to set a new master password ("Later" is allowed; the recovery code still works at the next start), done like "Change"; the recovery code keeps working. A slot the recovery code cannot open (one wrapped after a downgrade, without a recovery record) cannot be re-wrapped in this step;
  the old header is copied to `secret.key.mp.superseded-<timestamp>`, so remembering the old master password can still rescue it.
- **Forget**: "Forgot the master password?" in the unlock dialog → explain the consequences → confirm "Start a new key": the header, the list and the key file are renamed and kept (`.forgotten-<timestamp>`, not deleted; old keychain entries are untouched),
  a new local encryption key is started and the master password turned off. From then on it follows the first step's "entry not found" notice path: passwords are entered again when connecting; the plugin trust store cannot be read either and must be reset on the plugin manager page.
  If the password comes back to mind, renaming those files back restores everything (an automated test covers this path).
- **Audit**: turning on / off / changing (including setting a new one after recovery) / forgetting each records one `settings` audit entry (`master-password-enabled` / `-disabled` / `-changed` / `-reset`), the action only (whether a recovery code was created when turning on, the kept file names when forgetting), never any password material.

## 8. Migration and compatibility

- First step → second step: wrapping happens once, only when the user turns it on; nothing changes for those who don't (resolution, migration and fallback are byte-for-byte the first step's).
- Downgrading to a version that does not know `VSMP1`:
  - **macOS / Linux**: the wrapped entry does not match its id → the old version reports "not found", creates a new key and overwrites nothing; saved passwords cannot be decrypted in the old version. After upgrading again, that new plaintext key is not used while locked; unlocking with the master password wraps and merges it, and both old and new passwords decrypt.
  - **Windows** (and any case with only a key file, no keychain): the old version reads `DPAPI(VSMP1 ‖ records)` merely as "the key file is unusable", renames it to `secret.key.unreadable-<timestamp>` and creates a new key (no overwrite).
    After upgrading again the header matches no wrapped key → it is renamed to `secret.key.mp.orphaned-<timestamp>`, the app continues as off, and the message center explains it; passwords saved before the downgrade must be entered again.
    〔Known limitation〕The wrapped key is not recovered automatically from `.unreadable-*`: Windows has a single key-file slot that cannot hold two keys; all files are kept, and renaming them back by hand (after removing the `secret.key` the old version created) restores it.
- Multiple devices: the master password only covers this machine; the end-to-end passphrase of cloud sync is a separate thing, neither merged nor reused (so one leaked passphrase does not compromise both).

## 9. Decisions (settled by the maintainers on 2026-10-31)

1. When to unlock: one unlock dialog after the main window appears, cancellable; after cancelling, a lock entry stays in the status bar to unlock later.
2. Automatic locking: none by default; optional "lock after N idle minutes" and "lock when the system locks", wired up right away (the system lock can currently only be read on Windows; the option is not shown elsewhere).
3. Recovery code: turning it on creates a random 128-bit recovery code (skippable) that can also unwrap the DEKs (one more wrapping).
4. Plugin `ISecretsApi` while locked: the host first "waits for an interactive unlock"; cancelling maps to the closest meaning the current SDK can express (`PluginPermissionDeniedException`), never null;
   a separate "locked" contract is written in the SDK (end of §6), not used by the host in this round.
5. Windows Hello / Touch ID as an unlock method (item 2 of §8 in the [research](system-keychain-and-sudo-credential-fill-feasibility-research.md)): left for a third step, not done.
6. Argon2id fixed at m = 64 MiB, t = 3, p = 1, not lowered for low-end machines; parameters are stored in the header, and later tuning goes through rehash.

## 10. Tests

All run on Windows; the real-device macOS keychain / Linux Secret Service paths are only compiled in CI and exercised with the fake keychain (the real keychain round trip behind `VELASHELL_KEYCHAIN_TESTS=1` was not run).

- `MasterPasswordFormatTests`: round trips and rejections of the header, the wrapping and the recovery code (out-of-range parameters, truncation, duplicate keys, a record moved to another kind); while locked `Protect` creates no new key, `TryUnprotect` is false,
  and no temporary key is created when the keychain is unavailable; unlocking, the 1 / 2 / 4 … 30-second back-off after wrong entries, the recovery code (a typo does not count), zeroing on lock; an entry that unwraps but mismatches its id is not used; finishing an enable / disable that was interrupted;
  downgrade — a verbatim copy of the first step's resolver (a test fixture) reads the wrapped entries: no overwrite, a new key, and after upgrading again and unlocking both old and new decrypt; on the Windows path the header is renamed and kept.
- `MasterPasswordOperationsTests`: turning on / off / changing loses none of the DEKs when writing fails halfway, when the rollback also fails, and when the keychain is unavailable; a change broken off after adding records / after the header swap leaves a password that opens it, and leftovers are cleaned;
  setting a new master password after the recovery code, which keeps working; forgetting keeps everything and renaming back restores it; a header with old parameters is rehashed after unlocking; a settings import with a hand-written `masterPasswordProtection: true` changes neither the local value nor the actual state; one audit entry each, with no password material.
- `MasterPasswordLockedStateTests`: non-interactive never prompts, ten concurrent interactive requests show one dialog, all get false after a cancel; repository freezing (connections, shared credentials, the proxy password — including a copy decrypted before locking not being wiped, and filling it in after unlocking);
  Gist skips the round while locked instead of asking for a new token; plugin secrets are never read as null while locked, and deleting still works.
- Host: `MasterPasswordConnectFlowTests` (automatic reconnect while locked never prompts, says why and retries exactly once after unlocking; manual reconnect prompts; after a cancel the plaintext in memory is not used; connections without saved secrets are not gated),
  `MasterPasswordAutoLockTests` (no locking by default, the idle timer and its reset on unlock, no locking when the system state cannot be read, reading "not locked" for this session on Windows), `MasterPasswordDialogUiTests` (each dialog mode and the headless chrome spec),
  and two new `NotificationSourcesTests`.
