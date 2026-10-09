# Session Import

> Updated 2026-10-09. Three migration sources: **Xshell**, **WinSCP**, and OpenSSH **`~/.ssh/config`**;
> plus VelaShell's own connection files (**JSON / CSV** import and export, section 7, joesdu/VelaShell#571).
> 中文:[`../../zh/host/会话导入.md`](../../zh/host/会话导入.md)

Migrating from another tool happens exactly once, and it happens while the user still knows
nothing about VelaShell. So the default path through this dialog is **open it, glance at it,
click once** — not "first tell me which tool you are coming from".

## 1. What the dialog does

`SessionImportView` + `SessionImportViewModel` (`Presentation/ViewModels/`):

- **Scans every source on open**, presenting each as its own card. The user does not have to
  answer "import from where" first — the machine can look that up itself.
- **Fully automatic by default**: every supported, non-duplicate session is pre-selected and
  the primary button carries the count ("Import 12 sessions"), so one click finishes the job.
  Switch to "custom selection" to expand and tick individual rows.
- **Deduplicates across sources** on `host|port|user`: when the same machine is stored in both
  Xshell and WinSCP, only the first one scanned survives. Sessions that collide with one
  **already in** VelaShell are marked as duplicates too and left unticked (the "skip existing"
  switch can be turned off).
- **One group per source**: imported sessions land in a new group tagged with their source,
  rather than being mixed into the user's existing groups.
- **A result summary**: imported / passwords recovered / skipped.

When a source cannot be detected, its card offers a "browse" entry point — that is how portable
installations, non-default install locations, and config files kept elsewhere get in.

## 2. The three sources

| Source | Default location | Browse | Passwords |
| --- | --- | --- | --- |
| **Xshell** | The `Xshell\Sessions` directory under the `UserData` path in `HKCU\Software\NetSarang\Common\<version>`; falls back to `Documents\NetSarang Computer\{8,7,6,5}\Xshell\Sessions` | Folder | RC4 keyed by the current Windows identity (user name / SID, both the 5/6/7 and the 7/8 key schemes are tried). **Never attempted when a master password is enabled** — sessions still import, without passwords |
| **WinSCP** | Registry first, then the default `WinSCP.ini` | File | 0xA3 encoding plus a user-name/host-name prefix check. Master password as above |
| **OpenSSH config** | `~/.ssh/config` | File | **Nothing to recover** — an OpenSSH config file never stores passwords |

Half the code in the first two sources decrypts passwords. The third has none of that; all of
its difficulty is in the parsing rules.

## 3. How `~/.ssh/config` is parsed

`ssh_config` **is not an INI file**. It has no notion of "values scoped to a section": the same
keyword may appear in several blocks, and **block order** decides which one wins. Implement it
the way you would an INI file and it is correct for simple configs and wrong for every real one
that carries a catch-all block — wrong quietly, at that: the imported sessions all look right,
they just all log into the wrong account.

### 3.1 First value obtained wins

OpenSSH's rule is that **the first obtained value for each parameter is used**
(`ssh_config(5)`), which is why the customary layout puts named blocks first and the `Host *`
catch-all last, filling in only what the named blocks left out:

```sshconfig
Host prod
    HostName 10.0.0.5
    User deploy        # comes first, wins

Host staging
    HostName 10.0.0.6  # no User

Host *
    User fallback      # only staging gets it
    Port 2200          # both get it
```

Parsing uses the SSH library's `SshConfigFile` (the host does not keep a copy of its own): it keeps
the **whole block structure** (pattern list + ordered options), and `Resolve` walks every block
matching an alias in file order, keeping only the first value seen for each keyword. Harvesting
keywords while discarding which block they came from inverts this rule.

### 3.2 `Include` expands in place

Where an `Include` sits in the file **decides its precedence**: a `User` inside an include at
the top beats the catch-all further down the main file, and the reverse holds if the include
moves to the end. So expansion happens during parsing, not as "collect every file, then merge".

- Relative paths resolve against **`~/.ssh`** (OpenSSH's rule for the user config), not against
  the directory holding the config file — when the user browses to a config kept elsewhere, an
  `Include conf.d/*` inside it should still mean `~/.ssh/conf.d`
- Mutual includes are contained by a visited-file set plus a depth limit of 16 (the library's
  `SshConfigFile.MaxIncludeDepth`)
- Wildcard matches are **sorted** after expansion; otherwise the same config imports in a
  different order on a different file system

### 3.3 Which aliases count as "a machine"

Only **literal aliases** are imported. Blocks like `Host *` or
`Host *.internal !secret.internal` are templates that exist to back other hosts up; they are not
machines you can connect to. Their options reach the named aliases through the value rules,
which is precisely OpenSSH's semantics — no separate session should be, or needs to be,
produced for them. A negation (`!`) vetoes a match, so an excluded alias does not take that
block's options.

**`Match` blocks are evaluated by the library's rules, and do not apply when they cannot be
decided**: `all`, `host` and `originalhost` are evaluated as usual; `user` and `localuser` need to
know "as whom, and from which machine, we connect", `exec` needs to run a command, and
`canonical` / `final` we do not support — none of these can be decided during a static import, so
that block's options do not reach the sessions (negation is no different: `!exec` that cannot be
decided is still undecidable). They must not be treated as `Host *` — that would apply
conditionally-scoped options to every session unconditionally, and an internal jump host inside a
`Match exec "on-vpn"` would grow onto everything. A `Match` block never produces a session itself.
(Before 2026-09-25 the host used its own parser, which skipped every `Match` block wholesale, so
`Match all` / `Match host` did not apply either.)

When `HostName` is absent, **the alias is the host name** (with `Host build01` and no HostName,
ssh simply connects to `build01`). Without this, a config built on aliases plus a global `User`
would import nothing at all.

### 3.4 Field mapping

| ssh_config | VelaShell | Notes |
| --- | --- | --- |
| `Host <alias>` | Session name | Literal aliases only |
| `HostName` | Host | Defaults to the alias; `%h` expands to the original alias |
| `Port` | Port | Defaults to 22; out-of-range values ignored |
| `User` | User name | Defaults to empty (asked at connection time) |
| `IdentityFile` | Private key path + `AuthMethod.PrivateKey` | `~` and `%d` `%u` `%h` `%r` `%%` are expanded by the SSH library (`SshHostConfig.ExpandIdentityFiles`, the same expansion used when reading keys to connect; 〔History〕the host used to expand them itself, recognizing only `~` / `%d` and leaving `%h` in the path as-is); relative paths are resolved against `~/.ssh`. **Existence is not checked** — the key may not have been copied over from the other machine yet, and carrying the path into the profile is more useful than dropping it silently; `none` counts as unset |
| `ProxyJump` | Jump-host link (`JumpHostProfileId`) | See below |

**Every other keyword is ignored**: `LocalForward` / `RemoteForward` (port forwards do not
travel with a session import), `ServerAliveInterval`, `ForwardAgent`, and so on. They either
live somewhere else in VelaShell or the host does not support them yet — dropping them is
deliberate, because importing half a config invites worse assumptions than importing none of it.

### 3.5 Three judgement calls in `ProxyJump`

- **Take the last hop.** `ProxyJump a,b` means "through a to b, and from b to the target", so
  the hop nearest the target is **b**. VelaShell chains jump hosts one profile at a time (each
  profile points at its own previous hop), so taking the first hop reverses the chain. The hops
  further out are chained by a's and b's own profiles.
- **Resolve aliases within the current batch only.** Matching a config-local alias against the
  names of the user's existing sessions would chain two entirely unrelated machines together.
  When the jump host is unticked, or simply absent from this config, the session stays
  **direct** — it still imports, and the user can fill the link in from the connection dialog.
- **Break cycles.** If the config itself describes `a→b→a`, the later link is dropped: keeping
  it only makes the connection workflow's cycle detection throw, and a direct connection is the
  only usable shape left for those two sessions.

Parsing accepts `user@host:port` and bracketed IPv6; alias matching looks at the host part only.

## 4. Credential status in the preview rows

In custom-selection mode each row carries a credential status (four states, all five resx files
in sync):

| State | Meaning |
| --- | --- |
| Recovered | The password was decrypted; the imported session gets "remember password" and is re-encrypted with AES at rest |
| Password needed | The source holds a password that could not be decrypted (master password, a changed Windows account, …); the session still imports |
| **Uses key file** | The session authenticates with a private key (from `IdentityFile`) |
| No password | The source never stored one |

The third state arrived with SSH config support: telling the user "no password" about a session
configured with an `IdentityFile` is **misleading** — it is not missing a credential, its
credential is of another kind.

When a source has a master password enabled, the dialog shows one warning across the top:
**passwords in a master-password vault cannot be recovered; only the sessions come across**.
Unsupported protocols (WebDAV and the like) cannot be ticked and show their original protocol name.

## 5. Adding a source

1. Implement `ISessionImportService` (`Core/Import/`): `SourceKey`, `BrowseKind`,
   `DetectDefaultSource`, `ScanAsync`, `ImportAsync`
2. Write through `SessionImportWriter` (`Infrastructure/Import/`) — group creation,
   deduplication, password re-encryption, `IdentityFile` → key auth and `ProxyJump` → jump-host
   links all live there
3. **Append one line** of DI registration in `InfrastructureServiceCollectionExtensions`

The dialog needs no changes at all: it enumerates `IEnumerable<ISessionImportService>`.

## 6. Privacy and security

- All three sources are **read-only**; no external tool's configuration is ever modified
- `~/.ssh/config` is read **only when the user runs the session import** (what VelaShell reads
  otherwise under `~/.ssh` are keys and `known_hosts`, see
  [`PRIVACY.md`](https://github.com/joesdu/VelaShell/blob/main/PRIVACY.md))
- Recovered passwords are **re-encrypted with AES** by the repository at rest, and "remember
  password" is set **only when recovery actually succeeded** — a password that could not be
  decrypted does not leave an empty one behind pretending to be remembered
- Sources with a master password enabled are never attempted, rather than guessed at with the
  wrong key
- Connection files (section 7) carry no passwords by default; including them requires an export
  passphrase that encrypts the whole secrets section — **there is no option to export passwords in
  plain text**, and CSV never carries them

## 7. Connection files: VelaShell JSON and CSV (2026-10-09, joesdu/VelaShell#571)

The three sources above are about moving over from another tool; this section is about VelaShell's
own connection files — backups, moving to a new computer, handing a set to a colleague, and filling in
two hundred devices in Excel and importing them in one go.

Every entry point is in the explorer: "Import connections from file… / Export all connections… / Save CSV
import template…" in the "More" menu, "Export group… / Import into this group…" on a group's context menu,
and "Export selected…" on the multi-select menu (UI details in
[interaction-and-ui-specs.md](interaction-and-ui-specs.md) §3 and §12).

### 7.1 Two formats

| | VelaShell JSON | CSV |
| --- | --- | --- |
| Use | backups, moving to another computer, sharing | bulk editing in Excel, building a device list from scratch |
| Content | the complete connection (`SessionProfile` serialized as is) + its group + the shared credentials it references + port-forwarding tunnels | common fields: id, name, group, protocol, host, port, username, auth, password, private key, certificate, shared credential, jump host, tags, notes |
| Sensitive data | not exported by default; including it requires an export passphrase and encrypts the whole section (7.3) | never exported; an import may carry a `password` column |
| Encoding | UTF-8 without BOM | exported as UTF-8 **with** BOM (without it Excel opens the file in the local code page and every Chinese character turns into mojibake); import is covered in 7.4 |

An export covers the connections the user picked **plus their jump hosts** (all the way up the chain).
Exporting the machines behind a bastion without the bastion itself makes all of them direct connections
on the other computer — exactly the kind that cannot connect. The export dialog's summary has a separate
line saying how many were added this way.

### 7.2 JSON layout

```json
{
  "format": "velashell-sessions",
  "version": 1,
  "exportedAtUtc": "2026-10-09T08:00:00Z",
  "application": "VelaShell 0.9.0",
  "groups": [ { "id": "…", "name": "Production", "sortOrder": 0 } ],
  "sessions": [
    { "id": "…", "connectionType": "ssh", "name": "web-01", "host": "10.0.0.11", "port": 22,
      "username": "deploy", "authMethod": "privateKey", "privateKeyPath": "~/.ssh/id_ed25519",
      "groupId": "…", "jumpHostProfileId": "…" }
  ],
  "sharedCredentials": [ … ],
  "tunnels": { "<connection id>": [ … ] },
  "secrets": { "algorithm": "pbkdf2-sha256-200000+aes-256-gcm", "data": "…" }
}
```

- Enums are written as names (`"privateKey"`), and numbers are accepted on read; CJK text is written as
  is rather than as `\u` escapes — this file is meant to be read and edited by people
- Connections are `SessionProfile` serialized directly rather than a copied DTO: a copy is one more place
  where "remember to add the new field" can be forgotten, and the symptom is "a setting got lost after export
  and import". The price is that new fields go into the file automatically, so `SessionArchiveFieldTests`
  checks by reflection that every property is classified (configuration / secret / local state) and fails on
  any new, unclassified one — it guards against a new secret field being exported in plain text unnoticed
- Never in the plain part: passwords, key passphrases, plugin secrets (they go into `secrets` if exported at all),
  and the last-connected time (local state)
- A hand-written file may omit `format` and `id`: a `sessions` array is enough, and entries without an `id`
  are treated as new connections. Comments and trailing commas are allowed
- A `version` newer than this build is rejected outright rather than guessed at; a broken entry only breaks
  that entry (marked red in the preview), not the whole file

### 7.3 Sensitive data

- "Include passwords and other sensitive data" is unticked by default in the export dialog; ticking it
  requires an export passphrase of **at least 8 characters**, typed twice. **There is no "export passwords in
  plain text" option**, on purpose: exported files get mailed around and dropped into cloud drives, and a plain
  password written into one can never be taken back. The option is disabled when none of the connections has a
  saved password. CSV never carries sensitive data
- The encryption is the same as cloud sync's end-to-end passphrase: PBKDF2-SHA256 (200,000 iterations) derives an
  AES-256-GCM key, ciphertext Base64(salt16 | nonce12 | tag16 | cipher). Only the `secrets` section is encrypted;
  everything else stays plain
- On import, a file with `secrets` shows a passphrase box; **it can be imported without unlocking, just without
  passwords**. A wrong passphrase and corrupted data are not told apart ("Wrong passphrase, or the file is damaged.").
  An unknown `secrets.algorithm` (a future algorithm) counts as undecryptable: the file gets one notice and the
  connections still import
- Decrypted passwords are re-encrypted at rest by the target machine's repository with **its own** machine key

### 7.4 CSV columns and how they are read

The exported CSV doubles as the template; the header is always:

`id,name,group,protocol,host,port,username,auth,password,private_key,certificate,credential,jump_host,tags,notes`

| Column | Value | When empty |
| --- | --- | --- |
| `id` | connection id (filled in on export; matches the original exactly when imported back) | a new connection |
| `name` | display name | the host |
| `group` | group name | ungrouped |
| `protocol` | `ssh` / `sftp` / `ftp` (auto) / `ftp-plain` / `ftps` (explicit) / `ftps-implicit` / `plugin:<protocol id>` (an id with a dot may drop the `plugin:`) | `ssh` |
| `host` | host name or IP, **the only required column**. `user@host:port` and `[v6]:port` are split too, but only when the matching column is empty; a bare IPv6 address is not split. A user name / port split this way also applies when overwriting an existing connection (even without those columns) | the row is an error |
| `port` | 1–65535 | SSH / SFTP 22, FTP 21 (implicit FTPS 990); required for plugin protocols |
| `username` | user name | asked when connecting |
| `auth` | `password` / `key` / `certificate` / `agent` | certificate if one is given, key if a key is given, password otherwise. Always written on export (even when a shared credential is referenced), so a connection whose credential cannot be matched still comes back with its original method |
| `password` | password, not trimmed (always empty on export) | not stored |
| `private_key` / `certificate` | file path, not checked for existence | — |
| `credential` | shared credential name or id; when set, the connection references it and the `auth` columns do not count. When the name is not unique locally, the one the duplicate local connection already references is used; otherwise none | its own authentication |
| `jump_host` | jump host: another connection's name or id. Looked up in the file first, then on this computer; when the name is not unique locally, the one the duplicate local connection already jumps through is used, otherwise none (direct rather than wrong) | direct |
| `tags` | separated by semicolons (`,` and `\|` work too) | — |
| `notes` | notes, may span lines (wrap the cell in double quotes) | — |

Rows that give an FTP or plugin connection key / certificate / agent authentication are errors (those protocols
only take passwords). Reading is as forgiving as it can be:

- The delimiter is detected from the header row: comma, semicolon or tab (German-locale Excel saves semicolons,
  and cells copied out of a spreadsheet are tab-separated)
- Header names ignore case, spaces and `_ - .`, and common aliases plus CJK spellings are recognized (主机, 端口,
  用户名, 分组…; the table is `SessionCsvHeaderAliases`); column order does not matter, and unknown columns are ignored
  with a notice at the top. A file without a `host` column is rejected
- Encodings are tried in order: BOM (UTF-8 / UTF-16) → strict UTF-8 → the local ANSI code page. The last step is
  for Excel's default "CSV (Comma delimited)" on Chinese Windows, which saves GBK; when the regional format is not
  CJK but the UI language is, the UI language's code page is used
- CSV injection: on export, a cell starting with `= + - @` and the like gets a leading apostrophe so Excel treats it
  as text, not a formula; the apostrophe is removed on import, so nothing is lost on a round trip (names and notes
  may come from someone else's file)
- Problems in a row (empty host or a host with spaces, port out of range, unknown protocol) are marked with their
  line number in the preview without affecting the other rows; trailing empty rows from Excel are skipped

The template ships with two example rows using the documentation range 192.0.2.x (RFC 5737) and names starting with
`example-` — forget to delete them and the preview makes it obvious.

### 7.5 Duplicates and how things are written

- **What counts as a duplicate**: first the id (the same connection: a backup exported here, a CSV edited and
  imported back), then "protocol + host (case-insensitive) + port + username"; when the latter matches several,
  the one with the same name wins. Duplicates within the file are left alone — the user wrote them
- **When a connection already exists**, one of three, "Skip" by default:
  - **Skip**: the local one is untouched. Whatever in the file uses it as a jump host now points at the local one —
    exports carry jump hosts along, and on the other computer that jump host is usually already there, which is
    exactly the most common case
  - **Overwrite**: the id stays. JSON replaces the whole connection, but **the local password and passphrase are kept**
    when the file carries none (not exported or not unlocked), and so is the last-connected time; CSV **only changes the
    columns present in the file**, an empty `password` cell means "leave it" (it is always empty on export), and settings
    CSV cannot express — terminal settings, plugin settings — are kept as they are. "Export CSV → change usernames in
    Excel → import back" depends on exactly this. A local connection is overwritten at most once; a second row in the
    file that matches it is saved as a new one. The row with the same id comes first — it is that connection — so a row
    matching only by host cannot take it, even if it appears earlier. When overwriting would change the host of a local
    connection that has a saved password / passphrase, its preview row warns that "the password saved here will be used
    for the new host": exactly right when a server changed its IP, but a tampered file could use it to send the password
    elsewhere
  - **Keep both**: the file's one is saved as a new connection (new id), and jump-host references to it inside the file
    are redirected too
- In the preview, duplicate rows cannot be ticked in Skip mode (they would not be written anyway, and a tickable row
  would suggest otherwise); rows with errors cannot be ticked in any mode
- **The file's ids are kept when possible**: when an id does not clash with a local one it is reused, so moving files
  between two computers that sync through the cloud still shows the same connection to sync rather than a copy
- **Groups**: by default the file's groups — JSON matches a local group by id, then by name (case-insensitive); CSV by
  name; when nothing matches the group is created after the existing ones, in file order, reusing the file's group id.
  "Put into" can pick a local group or "Ungrouped" instead, putting everything there; coming from "Import into this
  group…" preselects that group. Each preview row shows the group it lands in, and groups to be created are marked "new"
- **Shared credentials** (JSON): one with the same id or name locally is reused and **not overwritten**; only when there
  is none is it created (with its password if unlocked). When the reused local one lacks a password / passphrase (as a
  credential pulled by cloud sync without an end-to-end passphrase does) and the unlocked file has it, the missing items
  are filled in; what is already there is left alone. CSV only references existing local ones
- **Jump hosts**: when one cannot be found (not in the file, not on this computer) the connection becomes direct, with a
  notice; a loop is broken at one link
- Write order: shared credentials → groups → connections → tunnels. Connections reference the first two; the other way
  round, a failure halfway would leave connections pointing at groups that do not exist, and they would vanish from the
  explorer
- Compromises made while writing (jump host made direct, credential not found…) are listed one by one in a dialog after
  the import (up to 15); without any, a single toast reports the result
- The dialog cannot be closed while it is writing (Esc, Cancel and the title-bar × do nothing); it closes itself when
  done. If writing fails halfway the dialog stays open with the reason; part of the file may already be in the store, so
  whether the user retries or cancels, the explorer is reloaded once the dialog closes (the batch edit dialog does the same)
- Files are limited to 16 MB (a CSV of two hundred devices is a few dozen KB; anything larger is almost certainly the
  wrong file)

### 7.6 Where the code lives

The rules are pure functions in `Core/Import/`: `SessionArchiveBuilder` (export scope and content), `SessionArchiveJson`
(JSON reading/writing and secret encryption), `SessionCsv` (CSV reading/writing; header aliases in
`SessionCsvHeaderAliases`), `SessionFileText` (encodings) and `SessionImportPlanner` (matching and the write set).
`Infrastructure/Import/SessionArchiveService` (`ISessionArchiveService`) only loads data from the repositories and writes
the computed result back; the three dialogs' view models live in the host's `ViewModels/` (`SessionExportViewModel` /
`SessionFileImportViewModel` / `SessionBatchEditViewModel`).

## 8. Not done yet

**Export in the other direction**: writing VelaShell sessions back into `~/.ssh/config`. It
belongs with "`known_hosts` interoperability with OpenSSH"; see
[`feature-plan.md`](https://github.com/joesdu/VelaShell/blob/main/feature-plan.md) in the code
repository.
