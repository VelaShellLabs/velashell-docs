# 06 · Permission System

> **Pending MCP transfer policy:** SDK version TBD (replace at release) adds an approval-origin hint, not a permission grant. Host integration will continue after official package publication; the text below still describes released default confirmation behavior. Subsequent integration permits separate user opt-ins for external MCP uploads/downloads, both off by default, with independent host checks of identity and saved policy. Leases, cancellation and revocation remain. See [Collaboration §2.3c](collaboration.md#mcp-transfer-auto-approval).

Design goal: give users Android-level transparency and control over “what this plugin can touch”, with declaration first, a prompt on first use of dangerous permissions, narrowing of scope, revocation at any time, and an audit trail.

> Trust-model premise (read first): Broker is a **logical enforcement point**. It fully constrains “all access through the host capability APIs” (SSH sessions, credentials, terminal, and host UI, resources that exist only in the host process and therefore cannot be bypassed by a plugin). However, the plugin process itself is an ordinary OS process, and **direct local-file and network OS calls cannot be physically intercepted by Broker in v1**.
> Therefore, in v1, the meaning of the fs.local/network permissions is “a behavioral contract for compliant plugins + user awareness”. Defense against malicious code depends on signatures/sources and the OS sandbox roadmap (12). UI copy must not exaggerate the strength of v1 protection.

> **Current download enforcement (SDK 2.1.2):** The universal Broker flow on this page remains a blueprint, not a guarantee for all file access. New `StartDownloadAsync` uses a separate `DownloadPermissionLease` checking remote read/local write with trusted per-job confirmation of plugin, actual session, source, final destination and overwrite policy. Default no-overwrite and explicit replacement approval apply; upload/terminal/session-opening grants do not carry over. Choosing a save path or AI Bypass cannot replace host confirmation; missing prompt/window means rejection. Plugin-management revocation synchronously invalidates pending/active leases and requests cancellation, with a pre-commit recheck and redacted subsequent reads, without promising rollback of committed files. Legacy synchronous file APIs do not acquire universal authorization. A real SSH / GUI end-to-end flow has not been accepted yet; see [Capability APIs §2.2](07-capability-apis.md#22-independent-download-jobs-sdk-212) for the platforms and filesystems where safe commit is available (Windows fixed local NTFS, local filesystems on Linux / macOS).

### Current AI Plugin Tool Scope

Desktop AI local-file tools use `LocalPathScope` to constrain access to the authorized workspace selected for the parent turn. Read-only Subagents inherit that scope and the current `ISessionScope` without creating or switching SSH sessions. They receive no writing, deletion, movement, process startup, approval management or further delegation tools. Inherited MCP tools must pass both the parent turn's tool filtering and the read-only filter, retaining their approval wrappers; server read-only annotations do not prove the implementation has no side effects.

`delegate_subagent` can select a configured chat model, defaulting to the parent model at delegation time. Changing models does not expand tool permissions. Credentials are resolved and revalidated through the target provider configuration; task results do not record keys. Prompts and tool-read results may be sent to the selected model's provider, so model configuration also determines the data recipient.

These are plugin-level tool constraints, not a completed universal Broker or OS sandbox. MCP workspace binding sets only the process working directory. Minimal/None environment policies reduce environment inheritance without isolating filesystem, network or same-user privileges. See [Automation and AI](11-automation-and-ai.md) and [Actual threat boundaries](12-security-threat-model.md#13-actual-boundaries-of-read-only-subagents-and-external-mcp) for implementation and validation scope.

## 1. Permission Manifest (apiLevel 1)

### Normal Permissions (Granted on Installation, Listed on the Details Page)

| id | Granted capability |
| --- | --- |
| `ui.contributions` | Register/update contribution points, VelaUI surfaces, dialogs, and notifications (rate-limited; in-app message-center notifications come under it with no permission of their own, and users can mute a plugin on the plugin management page — contract in [SDK §3.7](../sdk/sdk-reference.md#37-posting-to-the-message-center-sdk-213), SDK 2.1.3, wired in the host) |
| `storage.private` | Read/write the plugin's own data directory and KV storage |
| `settings.own` | Read/write settings contributed by the plugin |
| `i18n.read` | Read the current language/region |
| `sessions.observe.basic` | **Redacted view** of the session list (session display name and connection status only, no hostname/username/port) |

### Dangerous Permissions (Prompt on First Use; Bold Means High Sensitivity, with a Red Warning Bar)

| id | Granted capability | Scope that can be narrowed |
| --- | --- | --- |
| `sessions.read` | Full session metadata (host, user, port, tags) and connection events | By session |
| `remote.files.read` | SFTP read through a host session: list/stat/read/download | By session; by remote path prefix |
| **`remote.files.write`** | SFTP write: upload/write/mkdir/rename/delete | Same as above |
| **`remote.exec`** | Open a dedicated exec channel on a session to run commands (non-interactive, not sent to the user's terminal) | By session |
| `terminal.read` | Subscribe to the terminal output stream, read screen snapshots and selections | By session |
| **`terminal.write`** | Inject input into the user's terminal (equivalent to typing on the user's behalf) | By session |
| `fs.local.read` | Read local files through the host API | By directory prefix (user selects the directory when granting) |
| **`fs.local.write`** | Write local files through the host API | Same as above |
| `network` | Access the network from the plugin process (declarative in v1, see the premise above) | Declared domain list, shown on the details page |
| `clipboard.read` / `clipboard.write` | Clipboard | — |
| `secrets` | Store and retrieve secrets in the OS credential store within the plugin's **own namespace** | — |
| `audio.playback` | Play audio through the host audio service | — |
| `notifications.system` | Send system-level (OS) notifications | — |
| `automation.rules` | Register automation triggers/actions and create rule suggestions | — |
| `ai.invoke` | Call the host AI gateway (the model and quota configured by the user) | By model tier |
| **`sessions.create`** | Start a new session using a saved connection configuration (credentials are not exposed) | By connection configuration item |

Principles:

- **Credentials are never granted**: no permission can read passwords, private keys, or API keys. `sessions.create` only means “ask the host to connect using configuration 3”; credentials remain in the host throughout.
- **Write is stronger than read, and authorization is independent**: read and write are separate. Granting write implies none of the other capabilities beyond what is explicitly granted.
- Undeclared permissions **cannot be requested at runtime** (as with Android): the call immediately returns `PermissionDenied` without showing a prompt. Adding a permission requires a new release; newly added permissions are highlighted during upgrade for incremental review (Chrome-style).

## 2. Authorization Flow and UX

```text
Plugin calls a capability → Broker.Demand(pluginId, permId, scope)
  ├─ Already granted (and scope covers it) → allow, record in audit log
  ├─ Permanently denied → immediately PermissionDenied (do not disturb the user)
  ├─ Denied for this session → PermissionDenied
  └─ Undecided → enqueue an authorization prompt (merge concurrent requests from the same plugin; show prompts one at a time on the UI thread)
        ┌────────────────────────────────────────────┐
        │  🖼 Image Viewer (acme, verified)           │
        │  Requested permission: Read remote files    │
        │  Scope: /var/www/** on session              │
        │         "prod-web-01"                       │
        │  Plugin explanation: %perm.reason%          │
        │  (provided by manifest)                      │
        │  [Just this time] [This session] [Always allow] [Deny] [Always deny]│
        │  ☐ Narrow scope… (change path prefix/switch session)              │
        └────────────────────────────────────────────┘
```

- The prompt is an **in-app modal** (an overlay dialog inside the main window, reusing the existing dialog style). Its title includes the plugin name, publisher, and signature-verification status. High-sensitivity permissions add a red warning bar and a second confirmation, such as `terminal.write`: “This plugin will be able to enter commands into your terminal.”
- Decision granularity: `Just this time` (one call), `This session` (within the app session), `Always` (persisted), `Deny`, and `Always deny`.
- **Caller semantics**: the original call remains suspended while the prompt is shown (with a 60s timeout; timeout counts as denial for this request). Plugins should treat permission failure as a normal branch. The SDK provides `ctx.Permissions.RequestAsync(...)`, allowing a plugin to proactively request permission at an appropriate UX moment. It still goes through the same Broker and cannot bypass it.
- To prevent “consent fatigue bombardment”, a denied request for the same plugin and permission does not prompt again for 5 minutes and is denied directly. Each plugin may have at most 3 pending prompts; overflow is denied directly.

## 3. Persistence and Management

`~/.velashell/plugins/permissions.json`:

```jsonc
{ "version": 1,
  "grants": [
    { "plugin": "acme.image-viewer",
      "pluginVersion": "1.2.0",          // Version recorded when permission was granted, used for incremental permission comparison
      "permission": "remote.files.read",
      "decision": "allowAlways",         // allowAlways | denyAlways
      "scope": { "sessions": ["prof-a3f2"], "pathPrefixes": ["/var/www/"] },
      "grantedAt": "2026-07-24T10:00:00Z" } ],
  "integrity": "<HMAC, key stored in OS credential store>" }
```

- Integrity verification failure (the file was modified externally) → invalidate all grants and prompt the user to authorize again (fail-closed).
- **Settings → Plugins → Permissions**: dual views, by plugin and by permission, in the Android style. Each entry can have its scope changed or be revoked. Revocation takes effect immediately (invalidate the Broker cache and cut off active streaming subscriptions).
- Uninstalling a plugin → delete all of its grants. If an upgrade adds new dangerous permissions, those permissions start undecided and are not inherited.

## 4. Auditing

- Ring buffer (the latest 512 entries per plugin + a global file log, 7-day retention by default): time, permission, scope, result (`granted/denied/prompted`), and call summary, with an option to redact paths.
- “Recent activity” timeline in the management page: for example, “14:02 read prod-web-01:/var/log/nginx/access.log”. This brings the Android permission-usage panel into the product. **Post-event visibility** matters as much as the pre-event prompt.
- Persistent status-bar indicator: whenever a plugin is using `terminal.*` / `remote.exec`, show a pulsing icon, similar to a phone's microphone/camera indicator dot. Clicking it goes directly to the audit page.

## 5. Broker Implementation Notes

- Singleton, located at `Infrastructure/Plugins/Permissions/`. Capability-service entry points **must** go through `Demand` (implemented in the capability implementation base class, so new capabilities cannot bypass it, with analyzer rules and a code-review checklist as two layers of protection).
- Decision cache: in-memory dictionary + revocation version. Do not query repeatedly on hot paths, such as every frame of a terminal-output subscription. Validate once when the subscription is established; revocation invalidates the subscription through the version number.
- Scope matching: normalize path prefixes (remove `..`, with case handling according to remote FS semantics) before comparison. Bind session scope to the connection configuration ID rather than the volatile sessionId.

## 6. Development Plan (This Work Item)

| Task | Description | Dependency | Estimate |
| --- | --- | --- | --- |
| B-1 | Finalize the permission model: ID list, tiers, scope structure, and error semantics (review gate, affecting all API signatures in 07) | P-3 | 2d |
| B-2 | Broker core: Demand pipeline, decision cache, revocation version, fail-closed behavior; pure-logic unit tests | B-1 | 3d |
| B-3 | Persistence: permissions.json + HMAC integrity + migration framework | B-2 | 2d |
| B-4 | Authorization dialog (five-language copy), queue and anti-bombardment controls, scope-narrowing editor | B-2 | 4d |
| B-5 | Permission management page (dual views, revocation, immediate effect) | B-3, B-4 | 3d |
| B-6 | Audit pipeline + recent-activity timeline + status-bar indicator | B-2 | 3d |
| B-7 | Analyzer/review rules: capability implementations must pass the static check for `Demand` | B-2 | 1d |

Acceptance: permission-matrix tests cover every dangerous permission in seven states, undeclared, undecided, one-time, session, always, denied, and after revocation, with behavior fully conforming to this document. Consent-fatigue safeguards and integrity fail-closed behavior have dedicated tests.
