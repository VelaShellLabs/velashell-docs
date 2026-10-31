# System-Keychain Credential Storage + sudo Prompt Autofill: Feasibility Research

> Date written: 2026-08-25　　Baseline code: `dev` @ `7c43cdd`
>
> Purpose: answer two questions — ① can credentials be handed to the system keychain on Windows / Linux / macOS for safekeeping;
> ② when the terminal shows `[sudo] password for xxx:`, can the user pick a credential and send it straight in, instead of typing the password by hand over a poor network.
> Conclusions about the repository are backed by `file:line` references; details of the three platforms' native APIs are marked **unverified** and must be confirmed by hands-on testing during implementation
> (an established convention in this repository: adapted APIs go by what testing shows, not by what someone remembers).

## 2026-10-15 implementation erratum (#628)

The remaining sections of this document keep the research context of 2026-08-25; they are not the current implementation contract. Current behaviour is defined by [Interaction and UI specs §5.2.1](interaction-and-ui-specs.md#521-fill-a-shared-password-628-2026-10-15-entry-shape-redone-2026-10-16):

- The shared-credential store and its encrypted storage already exist and are reused as they are; the list reads only summaries, and only the selected password is decrypted after a click. No separate ICredentialVault or system-keychain dependency is created.
- ~~**Only the password is filled; no Enter is appended**; the `password + "\r"` in §4 item 5 below does not apply.~~ **Since 2026-10-24 picking an entry also submits it**: one Enter follows the text, the two are queued together as a single write, and anything typed right after the pick lands after the submit instead of becoming part of the password; the Enter is the bytes a real Enter key press would send at that moment (CR LF when the remote has newline mode on), not a hard-coded `\r`. **Since 2026-10-31 Enter is pressed only on a recognised password prompt**: when the Command Palette's manual entry is opened on another line (an ordinary shell prompt, an editor), or OSC 133 shell integration reports the shell idle at its own prompt, it fills only (see the Enter condition in Interaction and UI specs §5.2.1). The first version supports only password-authentication entries and does not mix in private-key passphrases or verification codes.
- There is no default shortcut; the Ctrl+Shift+K suggested below is already used for clearing the screen and cannot be adopted.
- Detection has been extracted into `InteractivePromptDetector`; the IsSecretPrompt used to suppress completion also covers verification codes and private-key passphrases, so it cannot serve directly as the criterion for the password entry point — filling uses a separate, conservative detection.
- The ECHO state of the remote PTY cannot be known reliably; EchoSuppressor is only a filter on local output bytes, not evidence that echo is off. The "suggest filling only when echo is off" of §5 item 4 below cannot be guaranteed by the current client. A remote echo may still reach the screen, logs and recordings, so the UI must warn rather than promise that nothing leaks.
- Before sending, the original session / connection generation and the input context are re-validated; after a cancel or a switch, late results are not sent; nothing goes through the clipboard, TypedInput or sync broadcasting.
- **Correction of 2026-10-16**: the entry shape changed from "a button in the top-right corner + a light hint + a confirming picker" to "as soon as a prompt line is detected, a list pops up on the line below the cursor, and clicking a row fills it". Wording about "confirming" above and below should be read as "clicking" under the new shape; the "suggest filling only when echo is off" below does not hold either, and the risk notice is now permanently shown above the list.
- When this erratum was written, this research had no English counterpart; the current facts were synced to the English [interaction-and-ui-specs.md](interaction-and-ui-specs.md#521-fill-a-shared-password-628-2026-10-15-entry-shape-redone-2026-10-16).

## 2026-10-25 implementation erratum (phase one done)

Phases zero and one have landed (the first step of master password protection): on macOS / Linux the local encryption key is handed to the system keychain, while Windows keeps the DPAPI-wrapped key file.
The design of the second step (an optional master password) is in [master-password-design.md](master-password-design.md). Differences from the research below:

- **Interface shape**: the name is still `IMasterKeyStore` (`Core/Data`), but its shape is "`Load` / `Save` by entry id, synchronous, every failure returned as `Unavailable(reason)` instead of thrown",
  not the `TryLoad(out key)` / `IsAvailable` of §3.2. The fallback chain, migration and multiple keys all live in `Infrastructure/Persistence/MasterKeyResolver`;
  there are no `FileMasterKeyStore` / `InMemoryMasterKeyStore` classes as drawn in §3.1 (the resolver manages the local key file itself, and the in-memory implementation exists only in tests).
- **Windows does not use Credential Manager** (`CredWriteW`): it keeps `secret.key` wrapped with DPAPI (current user), which is already keychain-grade; the "main benefit is that it becomes visible / revocable in Credential Manager" of the Windows row in §2 is not done.
- **macOS**: `SecItem*` + a generic password in the login keychain, service `com.velashell.master-key`, account = entry id. Access control is opened to "any application" (the equivalent of `security add-generic-password -A`) —
  release builds are ad-hoc signed and the signature changes with every release, so the default access control would raise an authorization prompt on the first start after every update; this is how the "item ACL bound to the code signature" pitfall of the §2 table is handled.
  Opened up, it sits at the same level as the former 0600 file (readable by the user's own processes); what it stops is "walking off with the data directory". With a stable Developer ID signature it can go back to trusting only VelaShell.
- **Linux**: libsecret's `secret_password_lookupv_sync` / `secret_password_storev_sync` (the non-variadic variants), schema `com.velashell.MasterKey`, attribute `key-id`, in the default collection.
  No libsecret installed, no session bus, or a locked keyring whose unlock prompt was cancelled → fall back to the file, log it and post a message-center notice — item 3 of §8 ("fall back silently") is settled this way: **not silently**, one notice.
  The "unlock prompt cancelled" branch is not confirmed on a real machine: the libsecret documentation does not define what a lookup returns for a locked collection or a dismissed unlock prompt;
  if it returns NULL without a GError, the current implementation records "not found" — a misleading message and one more unlock prompt, with no data loss (noted in the 2026-10-31 review; to be checked by hand on GNOME).
- **Entry ids and the list**: the data directory gains `secret.key.ref`, which lists entry ids only and holds no key material; id = the first 128 bits of a domain-separated SHA-256 of the key, so an interrupted migration always writes to the same entry, and a key read back can vouch for itself against its id.
- **Migration and rollback**: "keep the file copy for one release cycle after migrating" from §3.1 is **not adopted** (keeping a plaintext copy means nothing was migrated). Migration is "write to the keychain → read back and compare → write the list atomically → delete the file",
  and the deletion first renames to `.deleting`, then zeroes and deletes; any failing step stops at "still works as before" and is retried at the next start. Rollback safety comes from **keeping several keys side by side** instead: decryption tries each key in turn (the GCM tag tells whether it is the right one),
  encryption uses the first, and no key is ever dropped automatically; the list records every id, a run in which the keychain is unavailable gets a fresh local key that is merged once the keychain is back, and a `secret.key` created by an older version is merged the same way after upgrading again.
- **Asynchrony**: the interface is synchronous; resolution happens once at start-up (once `App.Initialize` has built the container) on a background thread, so D-Bus round trips and keyring unlock prompts never block the UI thread (the approach of §3.1).
  **Added 2026-10-31**: with a proxy password saved, reading the settings at start-up used to wait for that key (`App.ApplyPersistedPreferences` takes the snapshot on the UI thread), so the promise did not hold there;
  now the startup settings snapshot is published without the proxy password (`ISettingsService.GetStartupSnapshot`) and the password is filled into that same snapshot in the background once the key is ready; outbound connections that need the proxy wait on `SecretsLoaded`, and reading or saving the settings waits for the fill too.
- **"Show it honestly"**: not a row on the settings page but system messages in the message center (key file replaced / a key missing from the keychain / keychain unavailable right now / never migrated and the keychain unusable),
  see [message-center-and-feed.md](message-center-and-feed.md) §4A; `MasterKeyStatus` carries the protection level and the reason, so a settings page can read them directly if it ever shows them.
- Troubleshooting escape hatch: the environment variable `VELASHELL_NO_KEYCHAIN=1` turns the system keychain off (keys already moved there are not lost and come back on the first start without the variable); every start logs one `[Secrets] Master key: …` line, without key material.

## 0. Conclusion first

Both are doable, but **they are two independent things** with different value and different risk, so they should be phased:

| | What | Benefit | Risk |
|---|---|---|---|
| **Phase zero** | Land only the abstraction layer (`ICredentialVault` / `IMasterKeyStore`) + a file fallback implementation | Platform implementations become pluggable slots; the three platforms do not block one another | Very low. **Zero behaviour change, zero platform code** |
| **Phase one** | Hand the existing **local master key** to the system keychain | Closes the gap on Linux/macOS where "the key lies in a file in plaintext" | Low. There is a fallback path, and users notice nothing |
| **Phase two** | A credential vault + a **non-focus-stealing hint bar** at password prompts; click it to pick a credential and fill it in (manual entry also possible) | Directly addresses mistyped passwords over poor networks | Medium. The overlay must not become a focus trap; a misdetected prompt must never lead to an automatic send |

Phase one is a pure back-end swap; phase two is the feature users can see. The reverse order also works (phase two does not depend on phase one),
but doing phase one first means the sudo passwords stored by phase two get keychain-level protection from the moment they land.

## 1. Current state (what the repository already has)

**There is already a complete at-rest encryption scheme; the only thing missing is where the master key is kept.**

- `ISecretProtector` (`src/VelaShell.Core/Data/ISecretProtector.cs`) has just two methods, `Protect`/`Unprotect`,
  and is the single funnel for every sensitive field in the repository: session passwords and private-key passphrases (`SonnetDbSessionRepository.cs:13`),
  the cloud-sync GitHub PAT and end-to-end passphrase (`SyncModels.cs:34,37`), and plugin secrets (`SonnetDbPluginDataStore.cs:16`)
  all pass through it.
- The only implementation, `AesSecretProtector` (`src/VelaShell.Infrastructure/Persistence/AesSecretProtector.cs`):
  AES-256-GCM, ciphertext format `enc1:base64(nonce‖tag‖ciphertext)`; the master key is 32 random bytes
  stored in a local key file. **This is where the platforms differ** (`:115-126`):
  - Windows: `ProtectedData.Protect(key, null, CurrentUser)` — wrapped by DPAPI before it is written, which is already "keychain level";
  - Linux / macOS: **the 32 bytes are written to the file in plaintext**, protected only by `chmod 0600`.
- Conclusion: on Windows this is effectively done already; **the real gap is on Linux and macOS**.

**A password-prompt detector already exists, with tests.**

- `TerminalTabView.IsSecretPrompt(line, typed)` (`src/VelaShell/Views/TerminalTabView.axaml.cs:337`):
  after stripping input already echoed at the end of the line, it decides by "ends with a colon (half- or full-width) + a password-type keyword"; the keywords include
  `password / passphrase / passwd / 密码 / 口令 / verification code / 验证码 / 认证码` (`:340-366`).
- The existing tests `tests/VelaShell.Tests/ViewModels/SecretPromptDetectionTests.cs` directly cover
  `[sudo] password for pi:`, `root@192.168.1.1's password:`, `Enter passphrase for key ...:` and `密码：`.
- Today it is used to "not pop up command completion on password-prompt lines", which is exactly the criterion phase two needs — **no separate heuristic has to be written**.

**An injection channel already exists too, and it never broadcasts by construction.**

- `SshTerminalBridge.SendRaw(byte[])` (`src/VelaShell.Terminal/SshTerminalBridge.cs:203`):
  programmatic injection into the single outbound write queue (the comment at `:22` explains why the channel must never be written concurrently);
  connection startup commands already go through it.
- Key property: sync input (typing into several terminals at once) hangs off the `TerminalEmulator.TypedInput` event
  (`src/VelaShell/Services/SyncInputCoordinator.cs:26`), and **`SendRaw` does not raise that event** —
  so a password filled through the injection channel is not broadcast to the other terminals in the channel. This holds structurally, not by an extra guard.
- Session recording records **the bridge's output** (`src/VelaShell/Services/SessionRecorder.cs:7`), not input;
  sudo does not echo, so the password does not end up in the recording (for the exception see §5).

## 2. Keychains on the three platforms: what is available

> All API details below are **unverified** and must be confirmed by testing on real machines during implementation (including return codes, entry size limits and when permission prompts appear).

| Platform | Mechanism | How to call it | Main pitfalls |
|---|---|---|---|
| **Windows** | Credential Manager (`CredWriteW`/`CredReadW`, `CRED_TYPE_GENERIC`; DPAPI underneath) | `LibraryImport("advapi32.dll")` | A single blob has a size limit (about 2560 bytes, unverified); the repository already uses DPAPI, so the gain is mainly "visible and revocable in Credential Manager" |
| **macOS** | Keychain Services (`SecItemAdd`/`SecItemCopyMatching`/`SecItemUpdate`/`SecItemDelete`, `kSecClassGenericPassword`) | P/Invoke into `Security.framework` | **The item ACL is bound to the code signature**: when the signing identity changes (a local ad-hoc re-sign, a new certificate), the system shows the authorization prompt again. This interacts with the repository's `codesign --deep` release shape and must be tested |
| **Linux** | Secret Service (`org.freedesktop.secrets`, provided by gnome-keyring / KWallet) | P/Invoke into `libsecret-1.so.0`, or implement the D-Bus calls directly | **It may not exist**: with no desktop session, a pure SSH login or a minimal distribution there is no keyring daemon, or the keyring is locked. A fallback is mandatory |

Three judgements:

1. **There is no suitable cross-platform NuGet package.** Existing libraries either cover only Windows or bind all three platforms with heavy dependencies.
   The repository's established preference is thin wrappers written in the repository rather than heavy dependencies (the plugin RPC is implemented directly instead of pulling in StreamJsonRpc; DynamicData was removed),
   and the same applies here: one ~80–120-line P/Invoke layer per platform + one unified interface is enough.
2. **This would be the repository's first non-Windows native interop.** Every existing `LibraryImport` targets `user32/dwmapi/kernel32`
   (`src/VelaShell/Views/Win32WindowChrome.cs` and others); there is no precedent on the macOS/Linux side, and CI has no matching build gate either
   (`.github/workflows/` only has `release.yml`, triggered by a release or manually). Cross-platform verification has to happen on real machines.
3. **Linux must be allowed to fail.** "Refuse to store credentials when the keychain is unavailable" would lock out users without a desktop environment;
   the right approach is to fall back to today's AES key file and state truthfully on the settings page which one is in use.

## 3. Recommended architecture (phase one)

### 3.1 Abstraction layers: the terminal sees only the credential vault; platform differences are locked in the bottom layer

**Two seams are needed, not one.** What the terminal wants is "give me the credentials usable for this host", not "open the system keychain" —
if the terminal talked to the keychain abstraction directly, the fallback path for Linux without a keyring would leak into the UI layer and turn into platform branches everywhere.

```
  Terminal / settings page / plugins
        │   Know only this layer: ICredentialVault
        │   List(scope) / RevealAsync(id) / Save(entry) / Remove(id)
        ▼
  SonnetDB stores entries + ISecretProtector encrypts/decrypts   ← exists, not a line changes
        │   where the master key comes from
        ▼
  IMasterKeyStore     ← platform differences exist only in this layer
        ├─ WindowsCredentialStore   (advapi32 / DPAPI)
        ├─ MacKeychainStore         (Security.framework)
        ├─ LinuxSecretServiceStore  (libsecret / D-Bus)
        ├─ FileMasterKeyStore       (fallback = today's behaviour)
        └─ InMemoryMasterKeyStore   (tests)
```

**The repository has walked this road before.** `VelaShell.Core.csproj` states: Core references no concrete SSH library;
SSH capabilities are reached through the neutral abstractions under `Ssh/`, and the concrete library exists only in Infrastructure — "when migrating from SSH.NET to
Tmds.Ssh, not a line of Core changed". The keychain is the same move: the interfaces go into `VelaShell.Core/Data/`
(next to `ISecretProtector`), the implementations into `VelaShell.Infrastructure/Persistence/`,
wired up in `InfrastructureServiceCollectionExtensions.cs:51`. The terminal and the settings page use no `#if` and never ask about the platform.

**What the contract must contain** (otherwise the abstraction leaks):

- `IsAvailable` + `BackendName` + protection level: when Linux falls back to the file, the settings page must be able to show **truthfully**
  which one is in use, and must not pretend it is equally safe. The point of the abstraction is to smooth over the callers, not over the facts.
- **Asynchrony**: unlocking a keyring shows a system dialog, and D-Bus is an inter-process round trip; either may block for seconds.
  So the bottom layer is `ValueTask<byte[]?> TryLoadAsync()`; but `ISecretProtector.Protect/Unprotect`
  are synchronous and called everywhere (`AesSecretProtector.cs:32` is a `Lazy<byte[]>`) —
  **the fix is to await once at startup, resolve the master key, then hand it to the protector**, rather than spreading async to every caller.
  If this is not fixed in the abstraction, the first call that touches the key on the UI thread will freeze the startup screen.
- **The fallback chain is part of the abstraction**: `probe availability → use it; unavailable → file fallback → record the reason`,
  handled by one composite implementation; each platform implementation only handles its own kind, and none needs to know the others exist.
- **Migration is idempotent**: the first time "the keychain is available but has no entry" is detected, move the file key onto it,
  and keep the file copy for one release cycle (so a rollback does not lose data).

**So the first step need not touch platform code at all**: land the two interfaces `ICredentialVault` + `IMasterKeyStore`,
`FileMasterKeyStore` (exactly today's behaviour, still DPAPI on Windows), `InMemoryMasterKeyStore` (tests),
the fallback chain and the wiring. **Zero behaviour change, zero platform code**; afterwards each of the three platforms adds one class, without blocking the others,
and the credential vault and hint bar above need not wait for real machines on all three platforms before work can start.


### 3.2 Hand only the master key to the keychain; do not store credentials one by one


```
1 keychain entry (velashell / master-key) ──► 32-byte AES master key
                                              │
      SonnetDB still stores enc1: ciphertext ◄┘
```

Reasons:

- `AesSecretProtector` and all of its callers **do not change a line**; only the source of `LoadOrCreateKey` changes;
- every read or write against Linux's Secret Service is a D-Bus round trip and may show an unlock dialog;
  storing credentials one by one would mean one prompt per opened session, whereas the master-key approach **reads once per process**;
- the existing semantics of cloud sync (`SyncCrypto`), import/export and "cannot be decrypted on another machine" all stay unchanged;
- there is a single revocation point: when the user deletes that one entry in the system credential manager, all ciphertext is invalidated at once.

The new interface is roughly `IMasterKeyStore { bool TryLoad(out byte[] key); void Save(byte[] key); bool IsAvailable { get; } }`,
with one implementation per platform + one file fallback implementation, wired up in
`src/VelaShell.Infrastructure/DependencyInjection/InfrastructureServiceCollectionExtensions.cs:51`.
Migration: on first start, if the keychain is available and has no entry, move the existing file key onto it and keep the file copy for one release cycle (so a rollback does not lose data).

## 4. Recommended architecture (phase two: sudo fill)

**Credential vault**: a set of named entries `{label, username, password, optional scope (host/session/group)}`,
stored in SonnetDB with values passing through `ISecretProtector` — the same protection as session passwords, nothing built from scratch.
`SessionProfile` already has `Password` / `PrivateKeyPassphrase` / `RememberPassword`
(`src/VelaShell.Core/Models/SessionProfile.cs:38,41,47`),
and the sudo password is most of the time **the login password**, so the vault should be able to reference the current session's stored password directly rather than force the user to enter it again.

**Interaction shape (finalised with the requester on 2026-08-25)**: no automatic fill, and no dialog that interrupts typing.
When a password prompt is detected, only a **non-focus-stealing small hint** floats over the terminal; whether to use it, and which entry, is entirely up to the user's click.

```
  [sudo] password for pi: ▁
  ┌────────────────────────────────────┐   ← floats on the line below the cursor, does not cover the current line
  │ 🔑 Pick a credential  Ctrl+Shift+K │   click / shortcut → expands the credential picker
  └────────────────────────────────────┘
```

1. **Hint bar**: reuse the command-completion popup's positioning — anchored to the bottom edge of `PlacementRect` (the cursor row) and opening downwards
   (`src/VelaShell/Views/TerminalTabView.axaml:168-174`), flipping above automatically when there is not enough room below.
   **The position cannot conflict by construction**: password-prompt lines never show command completion anyway (`IsInteractivePromptLine` at
   `TerminalTabView.axaml.cs:293`), so the area below the cursor is empty at that moment, and the two overlays are mutually exclusive.
2. **Never steal focus**: `Focusable="False"` has to be set all the way down through the container, the list and `ListBoxItem`
   (`TerminalTabView.axaml:191,198`). This is not fastidiousness — the comment there records an incident found in testing:
   once focus landed in the popup, the Win+Shift+S screenshot overlay took system focus without raising Deactivated, and after switching back every keystroke
   went into the already-hidden popup window, so the terminal appeared "completely dead". The hint bar **appears automatically**, so it must follow the same rule.
3. **The picker expands only after a click**: this step is an explicit user action, so it **may** take focus (the picker needs search/typing),
   and it hands focus back to the terminal when it closes (`_termControl.Focus()`). The list is sorted by scope: entries matching the current session/host come first.
4. **Manual entry is a first-class citizen**: the bottom of the picker always has a "Type it in" entry, using the existing
   `SecurePasswordBox` (`src/VelaShell/Behaviors/SecurePasswordBox.cs`, backed by `SecureString` so plaintext does not linger in managed strings);
   after typing, the user can choose "This time only" or "Save to the vault". **This path must always be available** —
   an empty vault, an unavailable keychain or a temporarily changed password must all still allow typing it in on the spot.
5. **Send**: `bridge.SendRaw(UTF8(password + "\r"))`, queued in one go. Before sending, check once more that the current line is still a password prompt;
   do not go through the clipboard.
6. **Shortcut equivalent**: `Ctrl+Shift+K` (tentative; checked for conflicts: the existing Ctrl+Shift combinations take
   N/C/V/T/F/L/Tab — `src/VelaShell/Views/MainWindow.axaml:32-39`,
   `KeyboardShortcutService.cs:70`) summons the picker directly without touching the mouse; over a poor network this path is the fastest.
   Pressing it on a non-prompt line is also allowed to summon the picker (the user knows what they are doing); only the hint bar does not pop up by itself there.

The benefit over a poor network comes from **queueing the whole string at once**: the existing write queue guarantees order and completeness (the comment at `SshTerminalBridge.cs:22`
explains why concurrent direct writes are not allowed), which is far more robust than a person typing it character by character. So "one click" is already enough; nothing needs to be automatic.

## 5. Risks and hard constraints

1. **A misdetected prompt = the password goes onto the command line.** `IsSecretPrompt` is heuristic: opening a configuration file containing `password:` in vim,
   or `cat`-ing a log, may put a matching line under the cursor. With automatic sending, the password would become a shell command —
   on screen, in the shell history, in the session recording. **Mitigation**: send only after the user explicitly confirms; check once more before sending that the current line is still a prompt.
2. **Do not use the clipboard.** "Pasting" a password through the clipboard leaves it in the system clipboard and the clipboard history (Win+V). Use `SendRaw` and leave the clipboard alone.
3. **Sync input is safe by construction — do not break it.** See §1: `SendRaw` does not raise `TypedInput`. If someone later moves the injection to
   `WriteInput`, the password would be broadcast to every terminal in the channel — this must be pinned down in a comment.
4. **The recording exception.** Output is what gets recorded, and sudo does not echo, so it is safe; but for prompts that do echo, such as `read` (without `-s`),
   the password would end up in the recording. The hint bar should only suggest filling while echo is off (the terminal's existing echo-suppression / `EchoSuppressor` facilities can serve as a reference).
5. **A macOS signature change** causes keychain entries to need re-authorization (§2); the release shape is settled as a real executable + `codesign --deep`,
   and users must be warned in advance of the release that changes certificates.
6. **Failures must be visible.** When the keychain is unavailable it falls back to the file, but the settings page must state the current protection level truthfully and must not pretend it is equally safe.

## 6. Alternatives evaluated along the way

They address the same pain point and can be mentioned in documentation, but they do not replace the implementation:

- `sudo -A` + `SUDO_ASKPASS`: hands password retrieval to a script on the remote side — it has to be deployed on **every server**, which does not suit "connect to anything" use;
- `NOPASSWD` sudoers: solves it once and for all but lowers the remote security level; it is operations policy, not a client feature;
- `sudo -S` fed from stdin: requires rewriting the user's command (`echo pw | sudo -S ...`), and the password ends up in the shell history — **not recommended**.

Conclusion: the client-side "pick a credential, inject it in one go" is still the most suitable of these.

## 7. Effort and phasing

| Phase | Contents | Rough estimate |
|---|---|---|
| Zero | Two interfaces + `FileMasterKeyStore` (today's behaviour) + `InMemory` + fallback chain + wiring + unit tests | ~150 lines; zero behaviour change; no real machines needed |
| One | `IMasterKeyStore` implementations for the three platforms + migration + unit tests | ~100 lines of P/Invoke per platform; needs real-machine verification; can be delivered in three parts |
| Two A | Credential vault model/storage/settings-page CRUD + resx in five languages | ~500 lines + 4 resx files kept in sync |
| Two B | Hint bar + picker + injection wiring + misdetection guards + tests | ~350 lines |

Phases zero and one are invisible to users and can ship independently; phases two A/B should ship together — shipping A alone only adds one more place to store passwords.
Worth noting: **phase two need not wait for phase one** — as long as phase zero's abstraction exists, the vault can run on the file fallback first,
and the three platform implementations can follow later without changing a line above them.

## 8. Open decisions

1. **How the vault relates to session passwords**: a separate set, or "reuse the current session's stored password by default, with saving a separate copy optional"? (Leaning towards the latter.)
2. **Should every use require a second confirmation** (master password / Windows Hello / Touch ID)? It would noticeably enlarge the implementation surface.
3. **When Linux has no keyring**: fall back to the file silently (leaning), or require the user to choose explicitly on the settings page?
4. **Should phase two be limited to SSH sessions**, or also support local terminals (ConPTY) and plugin-protocol terminals?
