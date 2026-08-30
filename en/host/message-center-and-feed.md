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
| Examples | Update available, product news, CVE advisories | Fingerprint change, session drop, transfer failure |
| Presentation | Bell badge + list, revisitable | Dialog / status bar / tab flash, gone once seen |
| Retention | Across restarts (SonnetDB `notifications`) | None (security events also go to the audit log) |
| Owner | `INotificationCenter` | `ISecurityAlertService` / individual panels |

## 2. The interface

The entry point is the bell in the sidebar footer (left of the plugins and settings buttons). When
unread messages exist, a 6px accent dot appears at its top-right — there is only 24px of space, too
little for a number, and "is there anything new" needs no more than a dot.

The panel is a 360px non-modal overlay **anchored bottom-left, next to the bell** (an overlay should
grow from the button that opens it).

- **Header**: bell + "Messages" + unread badge | Unread (toggle) · Mark all read · Clear · Close
- **Each message**:
  - Unread rows carry a 2px accent bar on the left with a primary-colour Medium title; read rows fall
    back to secondary colour and normal weight
  - First line: kind badge (News / Update / Security / Offer; `warning` and `critical` severities
    switch to an outlined warning colour) + relative time
  - Title (wraps), body (two lines max, then ellipsis)
  - Destination line: action label + `›`, and **external links also show the host name**
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

## 3. Feed contract (what the backend publishes)

> **An official implementation now exists**: [velashell-feeds](https://github.com/joesdu/velashell-feeds),
> deployed at <https://feeds.easilynet.top/feed.json>. It aggregates CISA KEV and NVD advisories and
> ships an admin console for placing announcements and ads (reusing the plugin market's identity
> service, with an administrator allow-list). Self-hosted feeds are still fine — the format in this
> section is the only contract between them.

Set an **https** address under Settings → General → Message center → News feed.
**Leave it empty and nothing is subscribed — not a single request is sent.** A terminal client quietly
phoning home on a timer is the kind of thing people get asked about in enterprise environments, so it
has to be switched on deliberately by the user or the deployment.

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

- The id looks like `update:1.4.0`. The same version is republished on every startup and de-duplicated
  by that id, keeping its read state; a genuinely newer version is a new id and lights up as unread.
- **Store (MSIX) builds skip it entirely**: the install directory is read-only and updates are managed
  by the Microsoft Store, so "go to About to update" would send the user to a page that can do nothing.

> This fixed something along the way: `General.CheckUpdatesOnStartup` previously had no consumer
> anywhere in the repository (settings audit R-01 judged it "the update service is not wired up" and
> hid it). It now genuinely decides whether the startup check runs, and is therefore visible again on
> the General page. Update channel and auto-download remain unwired and stay hidden.

## 5. Storage and limits

- Location: SonnetDB document collection `notifications`, a single `inbox` document (through `IAppDataStore`)
- **Kept across restarts**: an announcement still makes sense the next day; this is the opposite of the
  file-transfer panel, where a transfer's progress is meaningless by tomorrow
- A **200-item** cap, dropping the oldest by publication time; expired items are pruned both on publish
  and on load
- Neither a corrupted store nor a failed write affects the running session: an unreadable store counts
  as no history, and a failed write is retried on the next change

## 6. Settings

A "Message center" section was added under Settings → General (`AppSettings.Notifications`):

| Setting | Default | Notes |
| --- | --- | --- |
| Announce available updates | On | Post "update available" into the message center |
| News feed | **Empty** | https address; empty means no subscription and no network activity |
| Fetch interval (hours) | 6 | Minimum 1; the feed is also fetched once at startup |
| Receive promotional messages | On | Turning it off drops `kind = promotion` items; announcements and security news still arrive |

The timer ticks on a fixed half-hour, and the interval setting decides whether a tick actually fetches
— so changing the interval takes effect on the next tick, with no restart and no timer rebuild.

## 7. Not done yet

- **Plugin-published notifications**: the SDK's `IUiApi` only has `ShowPanelAsync` today. The host-side
  interface is already shaped for this (`commandId` navigation works for plugin-registered commands),
  but actually opening it to plugins means changing the velashell-plugin-sdk repository, extending the
  isolated-process IPC protocol, and going through the SDK release process — a separate batch.
- **OS-level notifications** (Windows notification centre / macOS notifications): everything is in-app
  for now.
- **Grouping and search**: with a 200-item cap, the "Unread" filter is enough for the moment.
