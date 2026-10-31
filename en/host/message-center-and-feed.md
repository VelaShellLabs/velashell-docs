# Message Center and News Feed

> Added 2026-08-30. The design of the sidebar bell, the JSON contract for the news feed, and the targeting rules.
> **If you are building the push backend, section 3 is all you need.**

## 1. What it is, and what it is not

The message center holds things worth **keeping and revisiting**: a new version is available, an
announcement or security item from a subscribed feed, and — later — promotional messages pushed from
a backend.

It does **not** collect runtime alerts. A changed host fingerprint raises a dialog on the spot; a
dropped session flashes its tab and writes the status bar. Those need to interrupt immediately and
already have homes; adding them to the list would only bury what actually needs reading. The boundary
is documented on `Core/Notifications/INotificationCenter`; read it before adding a content source.

| | Message center | Runtime alerts |
| --- | --- | --- |
| Examples | Update available, product news, CVE advisories, last run ended unexpectedly, local encryption key could not be read, external-edit drafts left by the last run | Fingerprint change, session drop, transfer failure |
| Presentation | Bell badge + list, revisitable | Dialog / status bar / tab flash, gone once seen |
| Retention | Across restarts (SonnetDB `notifications`) | None (security events also go to the audit log) |
| Owner | `INotificationCenter` | `ISecurityAlertService` / individual panels |

## 2. The interface

The entry point is the bell in the sidebar footer (left of the plugins and settings buttons). When
unread messages exist, a 6px accent dot appears at its top-right — there is only 24px of space, too
little for a number, and "is there anything new" needs no more than a dot.

The panel is a 360px non-modal overlay **anchored bottom-left, next to the bell** (an overlay should
grow from the button that opens it). **Dragging its header moves it anywhere inside the window**; it
is clamped back into view when it would leave, and the position is written on mouse-up
(`ui-layout/notification-panel`) so the next open comes back where you left it — sharing the
`PanelDragHandler` / `IDraggablePanel` pair with the file-transfer toast rather than a second copy.

- **Header**: bell + "Messages" + unread badge | Unread (toggle) · Mark all read · Clear · Close
- **Each message**:
  - Unread rows carry a 2px accent bar on the left with a primary-colour Medium title; read rows fall
    back to secondary colour and normal weight
  - First line: kind badge (News / Update / Security / Offer / System; `warning` and `critical` severities
    switch to an outlined warning colour) + relative time
  - Title (wraps), body (two lines max, then ellipsis)
  - Destination line, **justified**: host name pinned left, action label + `›` pinned right, and only
    **external links carry a host name**. The action sits on the right because the pointer is already
    there — the per-row delete button and the scrollbar both live on that edge — which makes the trip
    from seeing it to clicking it the shortest. Host names vary in length (in-app jumps have none at
    all), so letting one push the other around would leave the action starting at a different x on
    every row; pinning each to its own edge keeps both columns straight
- **The whole row is clickable** — it navigates and marks the message read. The click target is a
  transparent button rather than a gesture on a Border, so keyboard Tab reaches it and focus and
  disabled states follow Avalonia's button semantics.
  **The hover highlight sits on the row's outer container, not on that button, which spans only the
  content column** — put it on the button and the delete-key column does not change with it, splitting
  the row into a light half and a dark half whose boundary reads as a stray vertical divider.
- The empty state changes with the filter: "No messages" / "Nothing unread"

### Navigation

A message can carry one destination, one of two kinds, **in-app first**:

| Field | Behaviour |
| --- | --- |
| `commandId` | Runs through `ICommandRegistry.Execute` — the same registry the menus, command palette and shortcuts share, so the target cannot drift from the other entry points. Commands registered by plugins work too |
| `url` | Opens in the system browser, **https only** |

If the command is not registered (returns false), it falls back to the external link rather than doing
nothing.

"Update available" uses the in-app path: `commandId = app.settings.about` opens settings and lands
directly on the About page, so the user updates right there instead of being dropped on the settings
front page to go find it. Section targeting goes through the `SettingsSectionKey` enum rather than an
index, and `SettingsSectionKeyTests` pins the two together — inserting a page into the middle of the
section list without updating the enum fails the test outright instead of silently landing on the
neighbouring page.

**Manual updates on Linux (from 2026-10-08; the matching package directly since 2026-10-24)**: when About → Check for
updates finds a newer version with a package for the current platform, it opens, in the default browser, the download
link of the package that matches **how VelaShell is installed**, for that version; the user installs it manually once
downloaded. The installation kind is read only from the file-system traces the packaging scripts leave — no guessing the
distribution, and no `dpkg -S` / `rpm -qf` subprocess:

- `APPIMAGE` is set and the app directory is inside the `APPDIR` mount → AppImage (`VelaShell-<version>-<RID>.AppImage`);
- the app directory is `/opt/velashell` and there is a dpkg record (`/var/lib/dpkg/info/velashell.list`, or one with
  `:amd64` / `:arm64`) → deb; no dpkg record but an rpm database (`/var/lib/rpm` or `/usr/lib/sysimage/rpm`) → rpm;
  neither → unknown;
- anywhere else → the portable tar.gz (taken straight from the manifest).

The status includes the file name and the address, which remains available if the browser cannot start. Prerelease
deb / rpm names contain `~`, and GitHub attachment names have no dependable convention for such characters, so instead of
guessing it opens that version's release page; when the installation kind is unknown, the architecture unsupported or
there is no manifest, it opens the whole releases list as before (prereleases included, for the preview channel). Linux
still does not check whether the installation directory is writable, download packages automatically or replace the
running installation. Startup checks still only post a notification and never open the browser automatically.
The Windows and macOS update flows are unchanged.

**Changelog entry and fixed targets (2026-10-21)**: messages still navigate to About; reading one does not download an update or restart the app. About → Changelog first reads the latest stable release's `releases/latest/download/latest.json`. A stable version newer than the installed version shows its notes with “Update available”; otherwise show the installed tag's notes with “Current version.” When no newer stable release exists, reuse the response if latest equals the installed version; only read the installed version's manifest if its notes are needed and that response cannot be reused, using `releases/download/<tag>/latest.json`. The installed tag may be a prerelease; this is not current-version-first or parallel rendering. This entry neither reads nor changes preview update settings and does not replace the existing channel logic of About → Check for Updates or startup checks.

- Notes use static manifests only, without querying releases or falling back to the anonymous Release API. `latest.json` retains `version` / `tag` / `assets` and adds optional `releaseNotes`: complete Markdown `body`, ISO 8601 or null `publishedAt`, and release-page address or null `htmlUrl`. Notes and the installation candidate come from the same response. Notes remain readable without valid platform assets, but no update action is offered.
- “Update now” on writable Windows / macOS installations downloads **the currently displayed manifest snapshot** (tag, version, asset URL, checksum), without querying latest again or letting background discovery change the target. Completion only marks the download ready and waits for the user to choose restart to update. A download with no selected target reports a failure rather than readiness; if the package was deleted or altered before the swap, it is removed and the ready state dropped, so the next check or Update now downloads it again, while other restart failures keep the ready state for a direct retry. Closing the notes dialog leaves the download running; About takes over the same task's progress, failure state, and restart action. Do not replace a running task or a ready package: reuse the same target, or report a conflict for a different target.
- On Linux / read-only installations the notes dialog shows “Go to release page” and opens **the selected tag's release page**, not the generic list used by the Check for Updates path above (since 2026-10-25 only this repository's releases pages are allowed; a missing or failing manifest link falls back to this repository's releases list). The user downloads and installs manually. Microsoft Store builds can read notes, but updates remain Store-managed, with no in-app download/install action.
- **Historical attachments are a prerequisite**: reading an installed historical tag requires `latest.json` on that Release. A missing attachment reports current-version notes unavailable, possibly a local build or a version without a release notes manifest, without an anonymous API fallback. An existing manifest without `releaseNotes` retains the tag and shows an empty body. Failure to discover latest is reported separately while still trying the installed version's notes; it must not mean “no update.”
- The release workflow reads Release notes with authentication, writes the manifest, then calculates `SHA256SUMS.txt`. Editing a release or using `notes_only` only synchronizes notes in an existing manifest and its checksum, without rebuilding packages. A missing original manifest alone is rebuilt as `latest.json` from the hashes in `SHA256SUMS.txt` and the sizes of the assets on the Release (a missing package or checksum record for any platform still fails). A missing checksum file requires a full release first; `notes_only` cannot create it. With both absent (most likely while the build is still running) the job emits a notice and skips. Work for one tag is serialized, but the two attachment uploads are not atomic and may need a rerun after failure. The workflow must reach the default branch and generate / backfill the target attachments before the notes fields are available.

See [Interaction and UI specs §14](interaction-and-ui-specs.md#14-settings-panel-implemented-as-an-840740-dialog-200px-left-navigation--right-content) for full version matching, caching, safe rendering, and the publishing contract. The download-progress rules below also apply to downloads started from the notes dialog and continued on About.

**Download progress carries a speed (from 2026-10-14)**: while an update package is downloaded in-app on
Windows / macOS, the About page's status line shows `percentage · downloaded/total · current speed`; when the
server does not give a total size it shows `N downloaded · current speed` instead, without a percentage. Until
the first progress report arrives (while connecting, or while verifying the prefix left by an earlier attempt)
it shows "Preparing update download…"; there are no bytes to speak of yet, so this time does not count as a
stall. The speed is measured over a sliding 3-second window (`DownloadRateMeter`) rather than averaged over the
whole transfer — when the network is fast early and slow later, the overall average explains neither "it's slow
now" nor "it was fast before". The window truly slides: it keeps the samples from the last 3 seconds, and the
speed is the byte delta from the oldest sample in the window to now divided by the elapsed time, so it does not
jump each time a new window starts; when the transfer resumes after stopping for longer than a whole window, the
first reading is 0 rather than the speed from before the stall. The download thread only reports when new bytes
arrive, so the UI also runs a 500 ms heartbeat (`UpdateDownloadTracker.Tick`) that refreshes between reports:
after the 1-second speed freshness window the speed is zeroed rather than leaving a stale number on screen, and
10 seconds without a single new byte is treated as a stall, switching the status line to "no data for N, the
connection may have dropped". All timing uses a monotonic clock (`Environment.TickCount64`, injectable for
tests), so a system clock correction or manual change cannot produce a false stall. The progress channel's type
changes from `IProgress<int>` to `IProgress<UpdateDownloadProgress>` (bytes downloaded, total bytes, speed).

The "stalled" text is only a hint; the actual reconnect is done by a watchdog in `UpdateService`. Silent drops
such as an expired NAT entry or a VPN going down do not make TCP report an error, so the read just hangs (the
download reads with `ResponseHeadersRead`, and `HttpClient`'s overall timeout does not cover the response body).
Each download attempt therefore gets an idle timeout that is pushed back on every progress report; **45 seconds
without progress** cancels the attempt and resumes from the `.partial` file with a Range request (segmented
downloads also keep per-segment progress in `.partial.meta`). Stall reconnects are capped at **3**; when they
are used up and still no data arrives, the status shows "Download failed. Please try again later.", the Check
for Updates button becomes clickable again, and the partial file is kept for the next attempt to resume. A
cancellation by the caller is not retried (`IUpdateService.DownloadUpdateAsync` now takes a `CancellationToken`
for this).

## 3. Feed contract (what the backend publishes)

> **An official implementation now exists**: [velashell-feeds](https://github.com/VelaShellLabs/velashell-feeds),
> deployed at <https://feeds.easilynet.top/feed.json>. It aggregates CISA KEV and NVD advisories and
> ships an admin console for placing announcements and ads (reusing the plugin market's identity
> service, with an administrator allow-list). Self-hosted feeds are still fine — the format in this
> section is the only contract between them.

The address lives under Settings → General → Message center → News feed and must be **https**.
It **ships pointed at the official feed** (`NotificationOptions.OfficialFeedUrl`): a security
advisory is worth something only if it arrives before the user goes looking for it, and a feed
that is off by default is one almost nobody ever turns on — the CISA KEV entries, the ones that
mean "someone is exploiting this right now", would reach no one.

The cost of that default has to be stated rather than hidden: a periodic outbound request now
happens **by default**, which is why it is written into the "Network connections" list in the
client's `PRIVACY.md`. **Clear the field and not a single request is sent** — a terminal client
quietly phoning home on a timer is the kind of thing people get asked about in enterprise
environments, so that exit has to stay open, and pointing the field at a self-hosted feed is
equally supported.

Outbound traffic uses the process-wide `HttpClient.DefaultProxy` (installed by `VelaWebProxy`), so it
automatically honours Settings → Network proxy, on the same path as update checks and Gist sync.

```json
{
  "schema": 1,
  "items": [
    {
      "id":          "2026-08-30-release-1.4",
      "kind":        "news",
      "severity":    "info",
      "title":       "VelaShell 1.4 is out",
      "body":        "Tunnel traffic statistics, automatic reconnect recovery, port-conflict precheck.",
      "publishedAt": "2026-08-30T02:00:00Z",
      "expiresAt":   "2026-10-01T00:00:00Z",
      "linkLabel":   "See details",
      "url":         "https://velashell.dev/releases/1.4",
      "commandId":   "app.settings.about",
      "locales":     ["zh-Hans", "zh-Hant"],
      "platforms":   ["win-x64", "osx-arm64"],
      "minVersion":  "1.2.0",
      "maxVersion":  "1.3.9"
    }
  ]
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `id` | ✅ | Stable identifier. **The key for de-duplication and read state** — republishing the same id is skipped and keeps the read state |
| `title` | ✅ | Truncated past 200 characters |
| `publishedAt` | ✅ | ISO 8601 (UTC). The list sorts by this, not by when it arrived |
| `kind` | | `news` (default) / `update` / `security` / `promotion` |
| `severity` | | `info` (default) / `warning` / `critical`; the latter two get a warning-coloured badge |
| `body` | | Truncated past 1000 characters, shown as at most two lines |
| `expiresAt` | | Hidden past this point, and pruned on the next load |
| `linkLabel` | | Action text; falls back to "Open" / "Read more" based on the destination |
| `url` | | **https only**; any other scheme is dropped (the item stays, it just loses its destination) |
| `commandId` | | In-app command id, takes priority over `url` |
| `locales` | | Targeting: UI language. Missing or empty array = unrestricted |
| `platforms` | | Targeting: RID (`win-x64` / `osx-arm64` / `linux-x64` …) |
| `minVersion` / `maxVersion` | | Targeting: version range (**inclusive**); pre-release suffixes are ignored in the comparison |

### Client-side hard limits (the backend must live within these)

- **At most 100 items per fetch**; the rest are dropped.
- **512 KB response cap**; anything larger abandons the fetch.
- **https only**: an announcement fetched in the clear can be swapped by a middlebox, and announcements
  carry links the user is going to click.
- **One bad record does not silence the whole feed**: items missing `id`/`title`/`publishedAt` are
  skipped individually and the good ones in the same batch still show.
- **Version targeting is skipped when the local version cannot be read** — better to show one item too
  many than to drop "this version has a problem, upgrade now", which is exactly the item aimed at you.
- An unreachable source, garbage response, or missing fields all count as "no new messages"; the user
  is never shown an error for it.

Every rule above is pinned by a test in
`tests/VelaShell.Core.Tests/Notifications/AnnouncementFeedDocumentTests.cs` — that file doubles as the
executable specification for this contract.

## 4. Local content source: available updates

Independent of the feed. On startup, subject to Settings → General → Check for updates on startup, a
check runs; if a newer version exists, a message is posted that lands on the About page.

- **Whether it checks**: "Check for updates on startup" is the network master switch — when off, not a single request is
  sent; when on, the check runs only if "Announce available updates" (message center section) or "Download updates
  automatically" is on — somebody has to want the result.
- The id looks like `update:1.4.0`. The same version is republished on every startup and de-duplicated
  by that id, keeping its read state; a genuinely newer version is a new id and lights up as unread.
- **Automatic download (since 2026-10-24, off by default)**: with "Download updates automatically" on and an installation
  that can update itself in place (a writable install directory on Windows / macOS; not Linux, the Store version or a
  read-only directory), a newer version is first downloaded in the background and verified against its SHA-256. It uses
  the same download path and the same busy state as About → Check for updates — progress shows if the About page is open,
  "Check for updates" does not start a second download, and the background download stands aside while the About page is
  already downloading; a package already prepared is taken as is.
  - When it is ready, `update-ready:<version>` is posted: title "VelaShell X is ready to install", action "Restart to
    update" (`app.settings.about`); it **never restarts by itself**. The id is separate from `update:<version>`, so an
    "update available" message already read lights up again.
  - When the download fails nothing pops up: it falls back to the plain `update:<version>`, and the About page's status
    line says it couldn't be downloaded in the background and will be retried at the next startup check; the user can also
    click "Check for updates" to try again.
  - With "Announce available updates" off and automatic download on, it still checks and downloads, it just posts nothing.
- The update item is posted **separately** from the feed and the crash notice: a download can take minutes, so the feed
  and the crash notice go first instead of waiting behind it.
- **Store (MSIX) builds skip it entirely**: the install directory is read-only and updates are managed
  by the Microsoft Store. About can display notes but cannot download/install an update, so no “go to About to update” reminder is posted.

> This fixed something along the way: `General.CheckUpdatesOnStartup` previously had no consumer
> anywhere in the repository (settings audit R-01 judged it "the update service is not wired up" and
> hid it). It now genuinely decides whether the startup check runs, and is therefore visible again on
> the General page. Update channel remains unwired and stays hidden; automatic download was wired on 2026-10-24 (settings
> audit R-03).

## 4A. Local content source: the local encryption key (since 2026-10-25)

The first step of master password protection handed the local encryption key to the system keychain (the macOS keychain / the Linux Secret Service; Windows
keeps the DPAPI-wrapped key file — see the 2026-10-25 erratum at the top of [system-keychain-and-sudo-credential-fill-feasibility-research.md](system-keychain-and-sudo-credential-fill-feasibility-research.md)).
The key is resolved once at start-up; when the result has something the user should know, a "System" message is posted instead of a start-up dialog —
for the same reason as the crash notice. These messages explain why saved passwords cannot be decrypted, and users usually come looking for that
after being asked for a password when connecting, so they are kept and can be read again. A successful move into the keychain and the normal Windows
case post nothing.

| Situation | id (de-duplication) | Severity | Title |
| --- | --- | --- | --- |
| The key file in the data directory could not be unlocked (another system account, a reinstall, copied from another computer); it was renamed and kept, and a new key created | `master-key:replaced:<kept file name>`, once | Warning | "The local encryption key could not be used and was replaced"; the body names the kept file |
| A key the data directory refers to is not in this computer's system keychain (data directory copied from elsewhere, entry deleted) | `master-key:missing:<ids…>`, once per set of ids | Warning | "A previous encryption key is not in the system keychain" |
| The system keychain could not be opened this time (keyring locked, no session bus, authorization refused) | `master-key:unavailable:<date>`, once a day | Warning | "The system keychain is unavailable right now"; the body says the passwords are not lost and are read as usual on the next start once it is back |
| Never migrated and the keychain is unusable, so the key stays in the data directory's key file | `master-key:file-fallback`, once | Info | "The encryption key is still stored in the data directory" |

- No action link: the way back is re-entering the password in the login dialog when connecting (existing behaviour).
- Posted in parallel with the update check, which may spend minutes downloading in the background, so these do not queue behind it; the periodic feed refresh does not post them again.

## 4B. Local content source: external-edit drafts (since 2026-10-25)

Each external-edit draft left by the last run (a session still open when the app crashed, a copy deliberately kept after a failed upload; see “Crashes and drafts” in [interaction-and-ui-specs.md](interaction-and-ui-specs.md) §6 for how drafts are told apart) gets one message:

- Kind "System", warning; id `remote-edit-draft:<subdirectory name>` — republished at every start and de-duplicated by that id, keeping its read state.
- Title "External edit not uploaded"; the body says which file, which path on which machine, and where the local copy is; an old copy without a state marker cannot name the remote path and says instead that it "may contain changes that were not uploaded".
- The action "Open folder" runs `commandId = files.remoteEditDrafts.open` (the command palette has "Open external edit drafts folder" too): with a single draft it opens that subdirectory, otherwise the `remote-edit` root,
  with each draft's full path in its body. A message-center action is just a command without parameters, so a draft cannot be re-uploaded from here — that needs a connection to that server first, and the session went away with the last run.
- The publication time is when the draft was first found, not its modification time: the list is sorted by publication time and drops the oldest past 200, so a draft from months ago would sink to the bottom or be pushed out right away.
- Once a draft has been deleted (the user dealt with it), its message is removed at the next start. Posted only at start-up (the directory scan runs on the thread pool); a draft left during a run is reported in the file panel on the spot and listed at the next start.

## 5. Storage and limits

- Location: SonnetDB document collection `notifications`, a single `inbox` document (through `IAppDataStore`)
- **Kept across restarts**: an announcement still makes sense the next day; this is the opposite of the
  file-transfer panel, where a transfer's progress is meaningless by tomorrow
- A **200-item** cap; beyond it plugin messages are dropped first (oldest by publication time first), and the
  host's own items go only once no plugin item is left. Each plugin additionally keeps at most 20, beyond which its
  own oldest is dropped (since 2026-10-31, so one chatty plugin cannot push out the host's one-shot notices);
  expired items are pruned both on publish and on load
- Neither a corrupted store nor a failed write affects the running session: an unreadable store counts
  as no history, and a failed write is retried on the next change

## 6. Settings

A "Message center" section was added under Settings → General (`AppSettings.Notifications`):

| Setting | Default | Notes |
| --- | --- | --- |
| Announce available updates | On | Post "update available" into the message center |
| News feed | **The official feed**, `https://feeds.easilynet.top/feed.json` | https address; swap in your own, or **clear it for no subscription and no network activity** |
| Fetch interval (hours) | 6 | Minimum 1; the feed is also fetched once at startup |
| Receive promotional messages | On | Turning it off drops `kind = promotion` items; announcements and security news still arrive |

The timer ticks on a fixed half-hour, and the interval setting decides whether a tick actually fetches
— so changing the interval takes effect on the next tick, with no restart and no timer rebuild.

## 7. Not done yet

- **OS-level notifications** (Windows notification centre / macOS notifications): everything is in-app
  for now.
- **Grouping and search**: with a 200-item cap, the "Unread" filter is enough for the moment.

In-app plugin notifications (`IUiApi.PostNotificationAsync` / `ui/notify`) are now wired into the message center. They support in-process and isolated plugins, plugin identity validation, at most six new messages per plugin per minute, persistent per-plugin muting, message sources and uninstall cleanup. See [SDK reference §3.7](../sdk/sdk-reference.md#37-posting-to-the-message-center-sdk-213) for the contract.
