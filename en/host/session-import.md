# Session Import

> Updated 2026-09-09. Three sources: **Xshell**, **WinSCP**, and OpenSSH **`~/.ssh/config`**.
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

`SshConfigParser` therefore keeps the **whole block structure** (pattern list + ordered
options), walks every block matching an alias in file order, and keeps only the first value
seen for each keyword. Harvesting keywords while discarding which block they came from inverts
this rule.

### 3.2 `Include` expands in place

Where an `Include` sits in the file **decides its precedence**: a `User` inside an include at
the top beats the catch-all further down the main file, and the reverse holds if the include
moves to the end. So expansion happens during parsing, not as "collect every file, then merge".

- Relative paths resolve against **`~/.ssh`** (OpenSSH's rule for the user config), not against
  the directory holding the config file — when the user browses to a config kept elsewhere, an
  `Include conf.d/*` inside it should still mean `~/.ssh/conf.d`
- Mutual includes are contained by a visited-file set plus a depth limit of 8
- Wildcard matches are **sorted** after expansion; otherwise the same config imports in a
  different order on a different file system

### 3.3 Which aliases count as "a machine"

Only **literal aliases** are imported. Blocks like `Host *` or
`Host *.internal !secret.internal` are templates that exist to back other hosts up; they are not
machines you can connect to. Their options reach the named aliases through the value rules,
which is precisely OpenSSH's semantics — no separate session should be, or needs to be,
produced for them. A negation (`!`) vetoes a match, so an excluded alias does not take that
block's options.

**`Match` blocks are skipped wholesale**: their conditions (`exec`, `originalhost`,
`canonical`) only have answers at connection time. Skipped rather than treated as `Host *` —
the latter would apply conditionally-scoped options to every session unconditionally, and an
internal jump host inside a `Match exec "on-vpn"` would grow onto everything.

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
| `IdentityFile` | Private key path + `AuthMethod.PrivateKey` | Expands `~` / `%d` / relative paths. **Existence is not checked** — the key may not have been copied over from the other machine yet, and carrying the path into the profile is more useful than dropping it silently; `none` counts as unset |
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

## 7. Not done yet

**Export in the other direction**: writing VelaShell sessions back into `~/.ssh/config`. It
belongs with "`known_hosts` interoperability with OpenSSH"; see
[`feature-plan.md`](https://github.com/joesdu/VelaShell/blob/main/feature-plan.md) in the code
repository.
