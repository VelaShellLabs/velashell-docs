# RDP plugin: design and use

中文: [RDP远程桌面插件设计与使用](../../zh/host/RDP远程桌面插件设计与使用.md).

`velashell.rdp` is an in-process workspace plugin in [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Rdp). It renders a remote desktop inside an Avalonia tab on Windows, Linux and macOS. The engine is a **pure-managed C# RDP client** — no FreeRDP, no native code — shipping one managed assembly for every RID.

## Architecture and scope

The protocol stack is the standalone C# library `VelaShell.Rdp` (MIT, released with the repository): it is UI-agnostic, does not depend on the plugin SDK, and implements the subset of MS-RDPBCGR / MS-RDPELE / MS-RDPECLIP / MS-RDPEDISP / MS-RDPEFS this plugin needs. The plugin references the library, whose DLL ships inside the plugin's signed `.vpx`; it is not published as a separate NuGet package. Layers:

- **Wire** — TPKT/X.224 framing, fast-path frames, ASN.1 PER/BER.
- **Security** — TLS via `SslStream` (OS trust, or an exact SHA-256 leaf-certificate pin); NLA/CredSSP via `NegotiateAuthentication` (NTLM + the TSRequest public-key binding).
- **MCS/GCC** — Connect-Initial/Response, erect/attach/channel-join.
- **PDUs** — client info, licensing, capability exchange, connection finalization.
- **Graphics** — interleaved-RLE decode and MPPC bulk decompression, a BGRA framebuffer; fragmented fast-path updates are reassembled.
- **Input** — fast-path keyboard/mouse/Unicode.
- **Channels** — clipboard (cliprdr) and drive mapping (rdpdr) over chunked static virtual channels; the dynamic virtual channel (drdynvc, MS-RDPEDYC) opens only the Display Control channel (MS-RDPEDISP) to negotiate dynamic resolution, and declines every other dynamic channel (audio, the GFX graphics pipeline) so the server keeps to the classic bitmap path.

The library's `RdpManagedSession` implements the `IRdpSession` / `IRdpClipboardTransport` contracts through an SDK-agnostic `ConnectAsync(RdpClientOptions, …)`; the plugin references the library, supplies the desktop view, input queue and clipboard bridge/codec, and maps the host's connect request to the library's `RdpClientOptions` while translating the library's exceptions into the host protocol exceptions. The host owns connection forms, credentials, certificate approval, session tabs and optional SSH jump transport; the plugin supplies the workspace and remote desktop. Connection, event pumping and teardown run asynchronously on the session worker task, interrupted by a cancellation token. The engine runs in-process and shares the host crash boundary; this is not process isolation or a filesystem sandbox. See [the SDK reference](../sdk/sdk-reference.md), [developer guide](../templates/dev-guide.md) and [current trust boundaries](../plugins/12-security-threat-model.md).

Implemented scope: desktop resolution that follows the window (dynamic resolution), fit/actual-size display, keyboard/mouse/Unicode input, Ctrl+Alt+Delete, in-view reconnect when a session drops, text/image/file clipboard, directory transfers and mstsc-style drive mapping (tick local disks or list custom folders). It does not negotiate audio, multiple monitors or drawing-order acceleration (RemoteFX/GFX); the server renders via classic RLE bitmap updates, with MPPC stream compression on weak links to cut bandwidth.

## Connect and use

Install the signed `velashell.rdp-<version>.vpx` from the plugin release using the host plugin manager, then create an RDP connection. Supply server, port (3389 by default), username and password. The optional domain is separate from the username. Choose an existing SSH session in **SSH jump host** when needed; the host manages the tunnel and the original destination remains the certificate identity.

| Setting | Default and accepted values |
| --- | --- |
| Domain (`domain`) | Empty; optional domain |
| Desktop width (`width`) | 1920; 320–8192 (initial size and fallback; follows the window once connected) |
| Desktop height (`height`) | 1080; 200–8192 (initial size and fallback; follows the window once connected) |
| Connect timeout (`timeout`) | 15 seconds; 1–120 |
| Security layer (`security`, advanced) | `auto` (default); `nla` NLA only; `tls` TLS only — see [Authentication and certificate policy](#authentication-and-certificate-policy) |
| Bidirectional clipboard (`clipboard`) | `false`; explicit opt-in for text, images and files |
| Local disks (`drive:<disk>`, **Drive mapping** page) | One checkbox per local disk, all unchecked by default |
| Custom folders (`drives`, **Drive mapping** page) | Empty; `name=absolute-path`, several separated by semicolons |
| SSH jump host (`jumpSession`) | None; host SSH-session selector |
| Trusted SHA-256 (`trustedThumbprint`) | Hidden setting supplied through certificate approval; exact leaf pin |

The remote desktop follows the workspace tab: on connect, and once a window resize settles (350 ms debounce), the plugin asks the server over the Display Control channel to match the desktop to the viewport in device pixels, so **Fit desktop** renders 1:1 and crisp. `width`/`height` are the size used on connect and the fallback when a server declines dynamic resize; then **Fit desktop** scales the mismatched framebuffer into the available area (high-quality interpolation) and **100%** shows actual pixels with scrollbars. Use **Ctrl+Alt+Delete** in the toolbar to send that sequence remotely. When a session drops, a **Reconnect** button appears over the desktop (the host session tab reconnects too); reconnecting renews credentials and certificate approval, and a failed attempt keeps the current view so the button stays usable.

### Clipboard and file transfer

Clipboard is **off by default**. Enabling it lets the remote desktop read and replace local clipboard content, including files.

- Remote text and images are received automatically. **Receive clipboard** explicitly receives remote files/directories: they are downloaded first, then their local file references are placed on the system clipboard for pasting into a file manager.
- **Send clipboard**, desktop focus and Ctrl+V publish local clipboard content. Unchanged local content is not repeatedly announced on focus, so it does not overwrite newly copied remote content. Files and directories use RDP clipboard file-content requests, independent of drive mapping.
- Downloads are retained beneath `<plugin DataDirectory>/rdp-clipboard`. They must remain present while file-manager paste operations still reference them. The 4 GiB retained-cache budget requires manual cleanup when full; clean up only files no longer needed by clipboard/paste consumers. Receiving files does not promise they land in a user's Downloads folder.

| Resource | Limit |
| --- | --- |
| Single clipboard payload | 16 MiB |
| Image dimensions | At most 8192 in either dimension |
| Decoded image pixels | At most 16 MiB, additionally to the encoded-payload limit |
| File descriptors, including directories | 4096 |
| One file transfer | 1 GiB total |
| Retained clipboard cache | 4 GiB |

Unsafe names, traversal and symbolic links are rejected. Directory structure is preserved within these limits. Clipboard transfers are plugin-managed RDP operations, not the host SFTP transfer queue or a grant to every host file API.

### Drive mapping

The connection dialog's **Drive mapping** page follows mstsc's *Local Resources → Drives*: one checkbox per local disk. On Windows every drive letter is listed, labelled by type — "Local Disk (C:)", "Removable Disk (E:)", "Network Drive (Z:)", "CD/DVD Drive (F:)"; on Linux / macOS the root (`/`), the home folder (`~`) and mounts under `/media`, `/run/media`, `/mnt` and `/Volumes`. All are **unchecked by default**.

Disks are enumerated from drive letters and types (or mount points) only — no media access, no volume query — so a disconnected network drive or an empty optical drive cannot slow plugin activation. Every 5 seconds the plugin checks the set of disks and re-registers the connection form only when it changed (with the same provider instance: open remote-desktop tabs are unaffected, and a dialog being edited keeps what was typed), so a USB disk plugged in later appears the next time the connection dialog opens. A ticked disk that is absent at connect time (USB unplugged, network drive offline) is skipped, not an error.

**Custom folders** on the same page share a single directory: enter `name=absolute-path`, separating several with **semicolons** (the host renders text settings as a single-line box, so there are no newlines). Examples: `work=C:\Work\Share;docs=D:\Docs` on Windows, `work=/home/alice/share` on Linux, `work=/Users/alice/share` on macOS. Line-separated values saved by earlier versions are still read. The always-visible hint under the box shows the format with examples for the current OS (the placeholder disappears as soon as anything is typed, so the examples live in the hint); a wrong entry fails the connection with a message that quotes that entry and the correct format, names a clashing name, or states the 32-mapping limit.

Names must be unique, and must not clash with a ticked disk's name (`C`, `E`, `root`, `home`, …); they contain **1–8 ASCII letters, digits or underscores**, hyphens not allowed. Disks and folders together allow at most **32 mappings**. Directories must already exist. The remote Windows desktop reaches a mapping as `\\tsclient\<name>`; an xrdp session at `~/thinclient_drives/<name>`. Each mapping grants remote **read/write access** to that disk or folder; nothing beyond the ticked disks and the listed folders is shared — no home directory, system disk or removable drive. Paths are confined to the mapping; traversal and symbolic links are rejected. Mapping a disk or directory and sending files through the clipboard are separate choices.

## Authentication and certificate policy

The security layer follows `security`: **Auto** (default) offers both TLS and NLA and lets the server choose (a server that supports NLA picks it); **NLA only** requires the server to select NLA and never falls back; **TLS only** offers TLS alone — the credentials travel in the Client Info inside the validated TLS channel and the server performs the logon itself (as when the server's "require NLA" setting is off). TLS 1.2 minimum; legacy RDP security and blanket certificate-ignore are disabled, and certificate validation is identical in every mode. Without a pin, verification uses OS trust plus server purpose, validity and hostname/IP, performed by .NET's `SslStream` against each platform's OS trust store.

**Auto-mode fallback.** If the server refuses right after the client's first CredSSP message — which is only the NTLM NEGOTIATE, carrying no user name and no credential proof — by answering with `errorCode` 0x80090302 (SEC_E_UNSUPPORTED_FUNCTION) or by aborting the TLS session (seen locally as SChannel 0x80090304 "The Local Security Authority cannot be contacted"; one server alternates between the two for byte-identical messages), it does not accept incoming NTLM. That is server policy, not a credentials problem. Auto then **reconnects once offering TLS only**, with the same certificate validation, and logs a warning in the plugin log. If the server refuses TLS too (RDP_NEG_FAILURE `HYBRID_REQUIRED_BY_SERVER`), it requires NLA yet refuses NTLM: the error says this is a server-side setting unrelated to the user name and password and gives both fixes — allow incoming NTLM on the server (Local Security Policy › Security Options › "Network security: Restrict NTLM: Incoming NTLM traffic" = Allow all), or turn off "Allow connections only from computers running Remote Desktop with Network Level Authentication". A wrong password (rejected after NTLM authentication) is still reported as an authentication failure and **never** triggers the fallback.

**Diagnosing connection failures.** Errors name the failing stage (TCP, security negotiation, TLS, NLA, MCS, logon, licensing, capability exchange) and the cause chain, including SChannel / socket error codes; failures after the TLS handshake also name the negotiated TLS version. When the server rejects the logon during NLA, its status code is turned into a specific reason (wrong user name or password, password expired, account locked or disabled, remote sign-in not allowed). When a **session is established and then dropped by the server** (a Set Error Info PDU, MS-RDPBCGR 2.2.5.1.1), its errorInfo code is turned into a reason shown in the disconnect overlay and the status bar: the user was signed out, another connection took over the session, the server denied it or the account lacks permission, an idle/logon timeout, no Remote Desktop licence available (expired RDS licensing or grace period), and so on; other codes fall back to a neutral description plus the hex code. The plugin log (`velashell.rdp`) records every failure with the full exception; failures after the TLS handshake add the cipher suite and the headers of the TLS records the server sent (type and length only, e.g. `Alert 26 B` — no payload is ever kept), which tells a server-side abort apart from a local decryption problem.

A host-approved pin is the exact SHA-256 hash of the leaf certificate's DER bytes (64 hexadecimal characters). **Pinning replaces CA/name/expiry validation** with a constant-time comparison of the leaf identity; it is not an additional condition layered onto those checks. A different leaf is rejected and requires a new explicit trust decision. SSH forwarding does not change the expected server name to loopback. Passwords go through APIs, never command-line arguments. NLA/CredSSP uses .NET's `NegotiateAuthentication` for NTLM and message sealing, with the TSRequest public-key binding per MS-CSSP.

## Building and independent delivery

The repository stays on .NET 11 and its SDK/Avalonia settings without changing dependencies. **Managed build only — there is no native step:**

```bash
dotnet build plugins/VelaShell.Plugin.Rdp -c Release
dotnet build plugins/VelaShell.Plugin.Rdp -c Release -t:PackVpx
```

The one platform-specific touch is `UnixClipboardFile`, which P/Invokes the OS `libc` (`open`/`fstat`) to open a clipboard file for local→remote transfer with `O_NONBLOCK|O_NOFOLLOW` (so a FIFO cannot block the open and a symlink is refused); Windows uses the .NET file API. The system libc ships with the OS and is not a bundled native library.

There is no longer an `rdp-native` workflow, CMake, bundled OpenSSL/VC runtimes or six platform native jobs: the plugin ships as **one managed `.vpx`** for every RID. CI and release run managed tests and packaging only. See the [plugin build README](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/README.md) and [packaging and signing](../templates/publishing.md).

## Verification and remaining boundaries

Unit tests cover settings, the clipboard codec, the headless UI and the CredSSP/NLA ASN.1 encoding. Live interoperability tests are gated on `VELA_RDP_TEST_TARGET` (skipped in CI) and exercise the full stack against a real server:

- TLS negotiation and leaf-certificate pin trust; without a pin a self-signed certificate is rejected and its SHA-256 is surfaced for the host trust flow.
- Desktop rendering: interleaved RLE **and MPPC bulk compression** (with the server's `bulk_compression` enabled, compressed PDUs decompress pixel-exact).
- Keyboard/mouse/Unicode input; text and image clipboard round-trips; drive mapping — ticked disks read remotely (including a listing of the system drive root `C:\`), custom-folder read and write.
- **Dynamic resolution**: a resize over the Display Control channel, validated as a 1024×768 ⇄ 1280×1024 round-trip; the framebuffer and status bar follow the reactivation.
- **In-host rendering**: the real `RdpWorkspaceView` and `RdpDesktopControl` render the engine's frames through Avalonia's headless Skia pipeline — the same control tree the host loads; the view is observed to resize the remote desktop to its viewport.

Besides offline ASN.1 tests, NLA/CredSSP is verified live against the FreeRDP shadow server (`/sec:nla`, gated on `VELA_RDP_TEST_NLA_TARGET`): NLA logon to the desktop; a wrong password reported as an authentication failure without fallback; TLS only refused by an NLA-only server with `HYBRID_REQUIRED_BY_SERVER`. The security-layer fallback and a first NLA reply that is a refusal (error code or TLS abort) are covered by in-process fake-server tests; every TLS 1.2 cipher suite a Windows server may pick (ECDHE/RSA × AES-GCM/AES-CBC) was exercised against xrdp. On real Windows servers, the behaviour of one that refuses incoming NTLM was confirmed (0x80090302 or a TLS abort on the first message); **a successful NLA logon against a real Windows server is still outstanding**. Interleaved-RLE / MPPC decode, MCS/GCC, capability exchange and the channel wire formats follow MS-RDPBCGR and the related published specs. These checks are not an all-platform or real-Windows-endpoint certification.
