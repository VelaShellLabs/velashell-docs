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
- **Channels** — clipboard (cliprdr) and shared folders (rdpdr) over chunked static virtual channels; the dynamic virtual channel (drdynvc, MS-RDPEDYC) opens only the Display Control channel (MS-RDPEDISP) to negotiate dynamic resolution, and declines every other dynamic channel (audio, the GFX graphics pipeline) so the server keeps to the classic bitmap path.

The library's `RdpManagedSession` implements the `IRdpSession` / `IRdpClipboardTransport` contracts through an SDK-agnostic `ConnectAsync(RdpClientOptions, …)`; the plugin references the library, supplies the desktop view, input queue and clipboard bridge/codec, and maps the host's connect request to the library's `RdpClientOptions` while translating the library's exceptions into the host protocol exceptions. The host owns connection forms, credentials, certificate approval, session tabs and optional SSH jump transport; the plugin supplies the workspace and remote desktop. Connection, event pumping and teardown run asynchronously on the session worker task, interrupted by a cancellation token. The engine runs in-process and shares the host crash boundary; this is not process isolation or a filesystem sandbox. See [the SDK reference](../sdk/sdk-reference.md), [developer guide](../templates/dev-guide.md) and [current trust boundaries](../plugins/12-security-threat-model.md).

Implemented scope: desktop resolution that follows the window (dynamic resolution), fit/actual-size display, keyboard/mouse/Unicode input, Ctrl+Alt+Delete, in-view reconnect when a session drops, text/image/file clipboard, directory transfers and explicitly mapped local directories. It does not negotiate audio, multiple monitors or drawing-order acceleration (RemoteFX/GFX); the server renders via classic RLE bitmap updates, with MPPC stream compression on weak links to cut bandwidth.

## Connect and use

Install the signed `velashell.rdp-<version>.vpx` from the plugin release using the host plugin manager, then create an RDP connection. Supply server, port (3389 by default), username and password. The optional domain is separate from the username. Choose an existing SSH session in **SSH jump host** when needed; the host manages the tunnel and the original destination remains the certificate identity.

| Setting | Default and accepted values |
| --- | --- |
| Domain (`domain`) | Empty; optional domain |
| Desktop width (`width`) | 1920; 320–8192 (initial size and fallback; follows the window once connected) |
| Desktop height (`height`) | 1080; 200–8192 (initial size and fallback; follows the window once connected) |
| Connect timeout (`timeout`) | 15 seconds; 1–120 |
| Bidirectional clipboard (`clipboard`) | `false`; explicit opt-in for text, images and files |
| Shared folders (`drives`) | Empty; explicit mappings only |
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

### Explicit disk/folder mapping

**Shared folders** starts empty. Add one `name=absolute-path` per line, for example `work=C:\Work\Share` on Windows, `work=/home/alice/share` on Linux or `work=/Users/alice/share` on macOS.

Names must be unique and contain **1–8 ASCII letters, digits or underscores**; hyphens are not allowed. At most **32 mappings** are accepted. Directories must already exist. The remote Windows desktop sees the mapping as `\\tsclient\work`. Each mapping grants remote **read/write access** to that folder. No home directory, system disk or removable drive is shared implicitly. Paths are confined to the mapping; traversal and symbolic links are rejected. Mapping a directory and sending files through the clipboard are separate choices.

## Authentication and certificate policy

TLS is negotiated, and NLA is used when the server requires it. TLS 1.2 minimum; legacy RDP security and blanket certificate-ignore are disabled. Without a pin, verification uses OS trust plus server purpose, validity and hostname/IP, performed by .NET's `SslStream` against each platform's OS trust store.

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
- Keyboard/mouse/Unicode input; text and image clipboard round-trips; shared-folder read and write.
- **Dynamic resolution**: a resize over the Display Control channel, validated as a 1024×768 ⇄ 1280×1024 round-trip; the framebuffer and status bar follow the reactivation.
- **In-host rendering**: the real `RdpWorkspaceView` and `RdpDesktopControl` render the engine's frames through Avalonia's headless Skia pipeline — the same control tree the host loads; the view is observed to resize the remote desktop to its viewport.

NLA/CredSSP is covered by offline ASN.1 tests; there is currently no NLA-capable test server, so **live NLA logon against a real Windows server is still outstanding**. Interleaved-RLE / MPPC decode, MCS/GCC, capability exchange and the channel wire formats follow MS-RDPBCGR and the related published specs. These checks are not an all-platform or real-Windows-endpoint certification.
