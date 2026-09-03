# Collaboration: chat bridge and outbound MCP server

> Owner: the AI assistant plugin (`plugins/VelaShell.Plugin.Ai`, id `velashell.ai`, shipped with the app).
> Entry point: command palette → **`AI: Collaboration`**.

This page covers the two routes that let VelaShell's agent work **alongside people and other agents**.
They point in opposite directions but share one security model (allowlists, modes, approvals), which is
why they are configured on the same page:

| Direction | Who uses it | Shape |
| --- | --- | --- |
| **Outbound** | your teammates | They @ the bot in Feishu / DingTalk / Telegram / WeCom; the agent works on an SSH session you already have open and posts the answer back to the chat |
| **Inbound** | other agents | Claude Code / Codex and friends call VelaShell as an MCP server, using the AI plugin's own toolset |

**Know the boundary first**: the bridge runs **inside the VelaShell process on your machine**.
Close VelaShell and the bot goes offline (turn on Settings → General → "minimise to tray on close"
to keep it resident). This is not a service you deploy on a server.

---

## 1. The chat bridge

### 1.1 Four channels, four inbound transports

| Channel | Inbound | Needs a public address | Can edit a sent message |
| --- | --- | --- | --- |
| Feishu / Lark | official long connection (WebSocket) | No | Yes (streams into one message) |
| DingTalk | Stream mode (WebSocket) | No | No |
| Telegram | Bot API long polling | No | Yes |
| WeCom | public callback (local HTTP listener) | **Yes** | No |

The first three **dial out from your machine**, so they work behind NAT with no networking setup.
WeCom only supports "the platform POSTs to an address you configure", which is why it is the one
channel that needs extra plumbing.

> **Feishu connection limits**: at most 50 long connections per app, and when one app runs several
> clients the platform delivers in **cluster** mode — one client picked at random, not a broadcast.
> **Do not run the same app credentials on two machines at once**; messages will land on the other
> one at random, which looks like "it sometimes doesn't answer".

### 1.2 Setting it up

**① Get the credentials** (each platform wants an app created in its own developer console):

| Channel | What to fill in | Where it comes from |
| --- | --- | --- |
| Feishu / Lark | App ID, App Secret | Open Platform → Credentials & Basic Info |
| DingTalk | Client ID (AppKey), Client Secret | Developer console → app credentials |
| Telegram | Bot Token | BotFather |
| WeCom | CorpID, AgentID, Secret, callback Token, EncodingAESKey | WeCom admin console → custom app |

Feishu needs two more things in its console, or **correct credentials still receive nothing**:
switch event subscription to the **long connection** mode and subscribe `im.message.receive_v1`,
then **publish a new version** of the app. These two are the most common way a Feishu setup fails.

**② Fill them in and press [Test].** The probe is read-only (exchange a token, read the bot's own
info, ask for the long-connection endpoint) — it sends no messages and changes no configuration.
It tells you separately whether the credentials are wrong, or the credentials are fine but the
endpoint was refused (which almost always means one of the two steps above is missing).

**③ Add the bot to a group**:

- **Feishu**: in the client, group settings → group bots → add bot, and search for your app name.
  (Feishu has **no** "scan to add the bot to a group" link — `applink`'s `client/app/open` opens a
  gadget page, and a pure bot app has none, so scanning it shows "page not available".)
- **Telegram**: once the test passes the settings page shows a QR code; scanning it jumps straight
  to the "add this bot to which group" picker.
- **DingTalk / WeCom**: follow each console's instructions to make the app visible to the target
  group or department.

**④ Authorise that chat.** While the allowlist is empty the bot **ignores everyone** — that is the
safe default: a bot that can run commands on production should not start obeying just because
someone dragged it into a group. Two ways, neither of which involves copying chat ids by hand:

- **Pairing code**: press "Generate" on the settings page and send `/pair 428913` in the chat you
  want to authorise. One use, ten minutes, five wrong tries and it dies.
- **One-click allow**: chats that tried to talk to the bot show up under "Chats that tried to talk
  to the bot", one row each with an "Allow" button.

> A pairing code **only adds a chat to the allowlist**. It cannot change the mode or the approval
> setting — it decides "may this chat talk to the bot", not "may it touch servers".

**⑤ Bind a server**: send `/sessions` in the chat to see what is connected, then
`/use user@host:port`. The binding stores `user@host:port` rather than a session id, so it survives
a reconnect and still works after VelaShell is restarted the next day.

### 1.3 What you can do from the chat

@ the bot in a group (in a direct message just talk). Slash commands:

| Command | What it does |
| --- | --- |
| `/help` | List these commands |
| `/sessions` | List the currently connected servers |
| `/use <user@host:port>` | Bind this chat to one of them |
| `/mode chat\|plan\|agent` | Change the mode (by default only downwards, see below) |
| `/new` | Drop the context and start a fresh conversation |
| `/stop` | Stop the turn that is running |
| `/status` | Show what this chat is set to |
| `/pair <code>` | Authorise this chat (available while unauthorised) |

### 1.4 Security model

These are deliberate, not accidents of the default values:

- **An empty allowlist means nobody is served.** The first time it is @-ed the bot replies once
  (and only once) with the chat id — silence would be "safer" but leaves the user stuck on step one,
  because the chat id is nowhere to be found in the Feishu or DingTalk UI.
- **The default mode is Plan (read-only)**, one notch more conservative than the chat panel.
  Someone is sitting in front of the panel and can stop it with a click; the person asking from a
  chat app may be on a train.
- **`/mode` can only lower the privilege by default.** Otherwise anyone on the allowlist could turn
  a read-only bridge into one that runs commands with a single message, and the "read-only" on the
  settings page would mean nothing. To open it, tick "Allow /mode to raise the mode from chat".
- **Approvals are text replies** (`y` allow / `n` refuse / `a` always allow in this conversation),
  and a timeout counts as **refused**. Interactive cards were not used because each platform's card
  callback has its own problems (Feishu's `card.action.trigger` is not reliably delivered over the
  long connection); text is the one channel that is solid on all four.
- **Approvers can be configured separately from "who may talk"**: anyone on duty may ask, but
  restarting a service needs the person in charge to say yes.

### 1.5 WeCom: why it needs one more step

WeCom can only push to a **publicly reachable callback address**, while this implementation binds
**`127.0.0.1` only** and offers no option to bind `0.0.0.0` — exposing a callback endpoint that can
run commands on production should not be something a checkbox decides.

To make it reachable: terminate HTTPS on a machine that has a public address (nginx or similar) and
forward that port to your local listener over a **reverse tunnel**. VelaShell has remote port
forwarding built in (Session → Tunnels → remote forward); point it at
`127.0.0.1:<the callback port on the settings page>`.

---

## 2. The outbound MCP server

Lets Claude Code / Codex / Cursor and friends call VelaShell's tools the other way round: enumerate
sessions, read the terminal, run one-shot commands, read and write remote files. The tools are the
same ones the AI plugin's agent mode uses, plus a `use_session` so the external agent can pick a
machine (call `list_sessions` first).

### 2.1 Connecting

Tick the switch and copy the whole "How to connect" box. Claude Code:

```bash
claude mcp add --transport http velashell http://127.0.0.1:8391/mcp \
  --header "Authorization: Bearer <token>"
```

Other clients take the generic `mcp.json` snippet (the second block on the settings page):

```json
{"mcpServers":{"velashell":{"type":"http","url":"http://127.0.0.1:8391/mcp",
 "headers":{"Authorization":"Bearer <token>"}}}}
```

### 2.2 Why HTTP and not stdio

stdio assumes the client can start the server, and VelaShell is a desktop application that is
**already running** — an external agent cannot start it, and should not start a second one just for
this. HTTP only asks the user to paste one address into their agent's configuration.

### 2.3 Security

- **Binds `127.0.0.1` only**, with no option to bind anything else.
- **Every request must carry the token**, which is generated randomly and cannot be turned off.
  Listening on loopback is not the same as being safe — any process on this machine, including a
  page in the browser, can post to a local port, and the token is the only door. The settings page
  can regenerate it at any time (remember to update your agent's configuration afterwards).
- **The default mode is read-only.** Usable out of the box is not the same as writable out of the box.
- **There is no approval UI on this route** — an external agent runs in another process, so VelaShell
  cannot show it an approval card. "Ask every time" therefore **refuses every write** here. To let it
  change things you must explicitly pick "read-only auto" or "bypass": a visible choice rather than a
  quiet default.
- "Allowed servers" restricts the external agent to a named set of machines.

This one is **on by default**: loopback-only plus a mandatory token plus a read-only default mode is
a low enough risk, while "off by default" only adds the pointless step of hunting through a settings
page before an agent can be connected.

---

## 3. When the machine is not connected, it can connect one itself

The agent on both routes used to be able to act **only on machines the user had already connected**:
if whoever was on duty closed that tab last night, a "is the production disk full?" in the group got
"connect one first" back. It now has three tools:

| Tool                  | What it does                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------- |
| `list_saved_sessions` | List the configurations **saved in the session tree** (including ones not connected right now); anything already connected comes back with its session id |
| `open_session`        | Connect to one of them                                                                                        |
| `close_session`       | Close a session **it opened itself**                                                                          |

### 3.1 The gate: it cannot connect to a machine you never saved

- The argument is a **configuration id**, not a host and port — which machines exist is something you
  decide in the session tree first;
- **Not one byte of credentials passes through the plugin.** Passwords, passphrases and fingerprint
  confirmations are all handled by the host's own dialogs;
- `close_session` **only closes what it opened**. It cannot touch the tabs you opened — an interface
  that can hang up a terminal somebody else is using should not exist.

### 3.2 Two confirmations, asking two different questions

`open_session` is the only tool that goes past two humans:

1. **This turn's approval** (reply `y`/`n` in the chat, press the approval card in the panel) — it
   asks "do we let it do this, this time";
2. **The host's confirmation dialog** (on your own desktop, showing the agent's **stated reason
   verbatim**) — it asks "do we let *this plugin* connect machines on my behalf".

The second one offers "always allow", and **that is what makes the unattended route work**: you
approve once at your desk, and from then on the bot in the group can connect by itself. To take it
back, revoke it on the plugin manager page (revoking there is all-or-nothing — the terminal-write
grant goes with it).

The reason is shown to you **verbatim**, never reworded. An agent that supplies no reason has the
call sent straight back to be rewritten — a confirmation dialog with no reason is just a button
people press blind.

### 3.3 How the modes relate to this route

- In **plan mode** `open_session` / `close_session` are not registered at all: planning means "say
  how you would do it first", and nothing should move at that step. `list_saved_sessions` is
  read-only, so it stays;
- The **outbound MCP route** has no approval UI, so `open_session` only works when "bypass approval"
  is selected ("ask every time" means refuse everything, and "read-only auto-allow" only clears
  commands with no side effects — opening a connection is not one). The server instructions mention
  this route only in that case: promising something that always hits a wall is worse than not
  mentioning it.

---

## 4. Known limits

- **WeCom needs a public entry point** (see 1.5).
- **Feishu card buttons** are not wired up: approvals are text replies.
- The bridge only handles **text messages**; images, files and rich text are not parsed yet.

---

## 5. Troubleshooting

| Symptom | Usually means |
| --- | --- |
| [Test] says "credentials OK, but the long-connection endpoint was refused" | Feishu: event subscription is not set to the long connection, or the new version was never published |
| @-ing it in a group does nothing | The chat is not on the allowlist (check whether the bot replied once with the chat id), or the bot was never added to the group |
| A 401 prefixed with `<provider> / <model>` | The **model provider** refused, not the chat platform — check that provider's key or sign-in state in the AI settings |
| An error starting with `Feishu … failed` | That one really is the chat platform |
| It answers sometimes and not others | The same Feishu app credentials are running on two machines (the platform delivers to one client at random) |

Switching models or signing in again **does not require restarting the bridge** — the AI settings are
re-read every turn, so the next message picks them up. Only channel credentials and allowlists need a
save on the settings page (which restarts the channels with the new configuration).
