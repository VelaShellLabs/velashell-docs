# RDP plugin: design and use

中文: [RDP远程桌面插件设计与使用](../../zh/host/RDP远程桌面插件设计与使用.md).

`velashell.rdp` is an in-process workspace plugin in [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Rdp). It renders a FreeRDP desktop inside an Avalonia tab on Windows, Linux and macOS. The implementation targets each OS's x64 and arm64 architecture; target coverage is not the same as verified interoperability.

## Architecture and scope

The engine is official FreeRDP **3.32.2**, pinned to commit `97eff6ddfbbfaa8e42a126d362686bd001ff8bdc`, under Apache-2.0. Its [official sample](https://github.com/FreeRDP/FreeRDP/blob/3.32.2/client/Sample/tf_freerdp.c) demonstrates client callbacks and GDI rendering for custom UIs. FreeRDP does **not** supply an official Avalonia component. A small C ABI bridge (`ABI 1`) owns FreeRDP; managed code copies BGRA frames into the Avalonia desktop view and translates keyboard, mouse and Unicode input. It does not embed an external client window.

The host owns connection forms, credentials, certificate approval, session tabs and optional SSH jump transport. The plugin supplies the workspace and remote desktop. Connection, event pumping and native destruction run on the session worker; abort is the cross-thread cancellation entry. Channel callbacks queue copied data, and shutdown completes before releasing native handles. Native faults share the host process; this is not process isolation or a filesystem sandbox. See [the SDK reference](../sdk/sdk-reference.md), [developer guide](../templates/dev-guide.md) and [current trust boundaries](../plugins/12-security-threat-model.md).

Implemented scope: fixed desktop resolution, fit/actual-size display, keyboard/mouse/Unicode input, Ctrl+Alt+Delete, reconnect through the host, text/image/file clipboard, directory transfers and explicitly mapped local directories. There is no audio playback/capture, multiple-monitor support or dynamic resolution. FreeRDP's internal discard-only audio backend is a channel setup dependency, not an audio feature.

## Connect and use

Install the signed `velashell.rdp-<version>.vpx` from the plugin release using the host plugin manager, then create an RDP connection. Supply server, port (3389 by default), username and password. The optional domain is separate from the username. Choose an existing SSH session in **SSH jump host** when needed; the host manages the tunnel and the original destination remains the certificate identity.

| Setting | Default and accepted values |
| --- | --- |
| Domain (`domain`) | Empty; optional domain |
| Desktop width (`width`) | 1280; 320–8192 |
| Desktop height (`height`) | 720; 200–8192 |
| Connect timeout (`timeout`) | 15 seconds; 1–120 |
| Bidirectional clipboard (`clipboard`) | `false`; explicit opt-in for text, images and files |
| Shared folders (`drives`) | Empty; explicit mappings only |
| SSH jump host (`jumpSession`) | None; host SSH-session selector |
| Trusted SHA-256 (`trustedThumbprint`) | Hidden setting supplied through certificate approval; exact leaf pin |

**Fit desktop** scales the fixed remote framebuffer to the available area; **100%** shows actual pixels. Neither negotiates a new remote resolution. Use **Ctrl+Alt+Delete** in the toolbar to send that sequence remotely. Reopen/reconnect through the host session tab when disconnected, so credentials and certificate approval can be renewed.

### Clipboard and file transfer

Clipboard is **off by default**. Enabling it lets the remote desktop read and replace local clipboard content, including files.

- Remote text and images are received automatically. **Receive clipboard** explicitly receives remote files/directories: they are downloaded first, then their local file references are placed on the system clipboard for pasting into a file manager.
- **Send clipboard**, desktop focus and Ctrl+V publish local clipboard content. Unchanged local content is not repeatedly announced on focus, so it does not overwrite newly copied remote content. Files and directories use RDP clipboard file-content requests, independent of drive mapping.
- Downloads are retained beneath `<plugin DataDirectory>/rdp-clipboard`. They must remain present while file-manager paste operations still reference them. The 4 GiB retained-cache budget requires manual cleanup when full; clean up only files no longer needed by clipboard/paste consumers. There is no automatic promise that receiving files puts them into a user's Downloads folder.

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

**Shared folders** starts empty. Add one `name=absolute-path` per line, for example `work=C:\Work\Share` on Windows or `work=/home/alice/share` on Linux. On macOS a typical path is `work=/Users/alice/share`.

Names must be unique and contain **1–8 ASCII letters, digits or underscores**; hyphens are not allowed. At most **32 mappings** are accepted. Directories must already exist; paths are resolved to their canonical directories, including resolving symbolic links. A mapping is not a filesystem sandbox. The remote Windows desktop sees the mapping as `\\tsclient\work`. Each mapping grants remote **read/write access** to that folder. No home directory, system disk or removable drive is shared implicitly. Mapping a directory and sending files through the clipboard are separate choices.

## Authentication and certificate policy

TLS/NLA are enabled, with TLS 1.2 minimum; legacy RDP security and blanket certificate-ignore are disabled. Without a pin, verification uses OS trust plus server purpose, validity and hostname/IP: CryptoAPI on Windows, SecTrust on macOS and OpenSSL's default OS trust paths on Linux. Windows chain policy includes revocation checking.

A host-approved pin is the exact SHA-256 hash of the leaf certificate's DER bytes (64 hexadecimal characters). **Pinning replaces CA/name/expiry validation** with exact leaf identity comparison; it is not an additional condition layered onto those checks. A different leaf is rejected and requires a new explicit trust decision. SSH forwarding does not change the expected server name to loopback. Passwords go through APIs, never command-line arguments. Native diagnostics can contain endpoint information.

## Building and independent delivery

The repository stays on .NET 11, SDK 2.1.5 and Avalonia 12.1.4. Managed compilation does not require staged native binaries, but runtime connections do. Linux/macOS clipboard file tests also require the real native bridge: it safely opens regular files without blocking on FIFOs or accepting special files. CI/release run managed tests on Windows after downloading the native artifacts; the six native jobs run CTest, not cross-platform managed tests. Native builds require Git, PowerShell 7, CMake ≥3.25, a C/C++17 compiler and OpenSSL development files; Linux additionally needs pkg-config, zlib and UUID development files. Build parallelism defaults to two and is capped at four. See the [plugin build README](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/README.md) and [native notes](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/native/README.md).

`Build-RdpNative.ps1` builds the pinned source, runs offline CTest checks, installs the bridge plus non-OS dependencies/notices and tests the installed shared library. The selected RID must match the native runner because tests execute there. Its default output is `artifacts/rdp-native/<rid>/package/runtimes/<rid>/native`; copy `runtimes` to the RDP project or set `-OutputDirectory` to that project. Generated binaries are excluded from Git. Unix library aliases are materialized as regular files before artifact upload so that Windows can package them. Windows includes discovered Visual C++ runtime files; macOS rewrites dylib load paths and applies ad-hoc signatures. System libraries remain OS prerequisites.

Both CI and release call the same reusable `rdp-native.yml`, using these native runners:

| RID | Runner / build baseline |
| --- | --- |
| `win-x64` | `windows-2025` |
| `win-arm64` | `windows-11-arm` |
| `linux-x64` | `ubuntu-24.04` |
| `linux-arm64` | `ubuntu-24.04-arm` |
| `osx-x64` | `macos-15-intel` |
| `osx-arm64` | `macos-14` |

Runner availability is documented by [GitHub](https://docs.github.com/en/actions/reference/runners/github-hosted-runners). These are configured build targets, not already-shipped artifact claims. Linux requires compatible glibc/libstdc++; macOS compatibility depends on the build deployment target. Alpine/musl and older operating systems are not implied supported.

The managed job waits for every native job, downloads and merges all six artifacts into the RDP project **before** `Bundle` / `PackVpx`, and validates the payload. Missing bridge, FreeRDP/WinPR, OpenSSL, `LICENSE.FreeRDP`, `NOTICE` or dependency license files fail packaging. All six RIDs travel in **one signed `.vpx`**, not six separate plugins. Native and managed release builds use the same release tag, including manually triggered runs. Local development can explicitly narrow `RdpNativeRids` to one staged RID; official workflows retain the six-RID default. See [packaging and signing](../templates/publishing.md).

FreeRDP's Apache-2.0 license and notice, and the licenses of bundled dependencies, are included per RID. The plugin uses the repository license; it does not relabel upstream code.

## Verification and remaining boundaries

The full plugin solution passed 882 tests with 196 skipped and no failures after restoring dependencies. The RDP project passed all 65 tests in Release with no skips, using the staged Windows native library. Coverage includes text, PNG/DIB pixel round trips, explicit remote file downloads, local file requests, simultaneous bidirectional transfers, input, cancellation and disposal. The packaging gate rejects missing required RIDs. The local unsigned `plugins/VelaShell.Plugin.Rdp/bin/vpx/velashell.rdp-2.0.7.vpx` development package contains Windows/Linux x64 only: all 57 entries were read through the SDK container API, with native dependencies and licenses present and no shared SDK/Avalonia or legacy IronRDP assemblies. It is not a signed six-platform release. Both language pages and their index links were checked locally.

Existing local evidence records Windows x64 and WSL Debian Linux x64 native builds, offline tests and installed-library ABI smoke tests. These cover buffers, lifecycle, channel/clipboard handling, rejection/pinning and loopback cancellation. The Debian build uses a newer glibc than the CI Ubuntu baseline and is not the release Linux artifact. The latest native test results remain recorded with the implementation.

macOS and ARM64 have **not been locally executed**. The new six-platform workflow must actually pass before its outputs can be called available. No real RDP endpoint was supplied: successful live TLS/NLA, remote desktop rendering, real clipboard transfers and mapped-folder interoperability still require endpoint tests on each OS/architecture. Offline tests and build configuration are **not real-server or all-platform certification**.
