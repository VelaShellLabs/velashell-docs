# RDP 远程桌面插件设计与使用

English: [RDP plugin: design and use](../../en/host/rdp-plugin-design-and-use.md)。

`velashell.rdp` 位于 [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Rdp)，是进程内工作台插件，在 Windows、Linux、macOS 的 Avalonia 标签页中显示 FreeRDP 桌面。构建目标覆盖各系统的 x64 与 arm64；目标覆盖不等于真实互操作已经验收。

## 架构与范围

引擎采用官方 FreeRDP **3.32.2**，固定提交 `97eff6ddfbbfaa8e42a126d362686bd001ff8bdc`，许可证 Apache-2.0。[官方示例](https://github.com/FreeRDP/FreeRDP/blob/3.32.2/client/Sample/tf_freerdp.c) 展示客户端回调与 GDI 绘制，可接入自定义界面；FreeRDP **没有官方 Avalonia 控件**。小型 C ABI 桥接层（`ABI 1`）管理 FreeRDP，托管端将 BGRA 帧复制进 Avalonia 桌面视图，转换键盘、鼠标与 Unicode 输入，不嵌入外部客户端窗口。

宿主管理连接表单、凭据、证书批准、会话标签与可选 SSH 跳板；插件提供工作台和远程桌面。连接、事件循环与原生销毁在会话工作线程完成；abort 是跨线程取消入口。通道回调复制数据入队，关闭完成后才释放原生句柄。原生故障与宿主处于同一进程边界，不是进程隔离或文件系统沙箱。相关契约见 [SDK 参考](../sdk/sdk-reference.md)、[开发指南](../templates/dev-guide.md) 和 [当前信任边界](../plugins/12-security-threat-model.md)。

范围包括固定桌面分辨率、适应窗口/实际尺寸、键鼠/Unicode、Ctrl+Alt+Delete、通过宿主重新连接、文本/图像/文件剪贴板、目录传输及显式本地文件夹映射。不提供音频播放/采集、多显示器或动态分辨率。FreeRDP 的内部丢弃式音频后端仅服务通道初始化，不代表音频功能。

## 连接与操作

通过宿主插件管理页安装插件 Release 中已签名的 `velashell.rdp-<版本>.vpx`，新建 RDP 连接，填写服务器、端口（默认 3389）、用户名和密码。域为独立的可选字段。需要跳板时，在“SSH 跳板机”选择已有 SSH 会话；隧道由宿主管理，证书仍核验原始目标身份。

| 设置 | 默认值与允许范围 |
| --- | --- |
| 域（`domain`） | 空，可选域 |
| 桌面宽度（`width`） | 1280；320–8192 |
| 桌面高度（`height`） | 720；200–8192 |
| 连接超时（`timeout`） | 15 秒；1–120 |
| 双向剪贴板（`clipboard`） | `false`；明确开启后支持文本、图像与文件 |
| 共享文件夹（`drives`） | 空，只共享显式映射 |
| SSH 跳板机（`jumpSession`） | 无，由宿主 SSH 会话选择器提供 |
| 已信任 SHA-256（`trustedThumbprint`） | 隐藏设置，由证书批准流程提供，精确叶证书 pin |

“适应窗口”把固定远程帧缓冲缩放到可用区域；“100%”显示实际像素，二者都不改变远程分辨率。工具栏“Ctrl+Alt+Delete”将组合键发往远程。断开后从宿主会话标签重新打开/连接，以更新凭据及证书批准。

### 剪贴板与文件传输

剪贴板**默认关闭**。开启后远程桌面可以读取及替换本地剪贴板，包括文件。

- 远程文本和图像自动接收。“接收剪贴板”显式接收远程文件/目录：先下载，再把本地文件引用放进系统剪贴板，供文件管理器粘贴。
- “发送剪贴板”、桌面取得焦点及 Ctrl+V 发布本地内容。本地内容未变化时不因取得焦点而重复发布，避免覆盖刚复制的远程内容。文件和目录通过 RDP 剪贴板文件内容请求传输，与磁盘映射独立。
- 下载保留在 `<插件 DataDirectory>/rdp-clipboard`。文件管理器粘贴仍引用它们时必须保留。累计缓存上限为 4 GiB，满后需手动清理；只删除已不再供剪贴板/粘贴使用的文件。接收操作不意味着文件已经放入用户的“下载”目录。

| 资源 | 上限 |
| --- | --- |
| 单次剪贴板载荷 | 16 MiB |
| 图像尺寸 | 任一方向最多 8192 |
| 解码图像像素 | 额外限制为 16 MiB，不能只满足编码后载荷上限 |
| 文件描述符（含目录） | 4096 |
| 单次文件传输 | 合计 1 GiB |
| 保留的剪贴板缓存 | 4 GiB |

拒绝不安全文件名、路径穿越和符号链接。在这些限制内保留目录结构。剪贴板传输由插件管理 RDP 通道，不是宿主 SFTP 传输队列，也不代表授予全部宿主文件 API 权限。

### 显式磁盘/文件夹映射

“共享文件夹”初始为空，每行写 `名称=绝对路径`。Windows 示例：`work=C:\Work\Share`；Linux：`work=/home/alice/share`；macOS：`work=/Users/alice/share`。

名称必须唯一，只允许 **1–8 位 ASCII 字母、数字或下划线**，不允许连字符；最多 **32 个映射**。目录必须存在；路径会解析为规范目录，包括解析符号链接。映射不构成文件系统沙箱。远程 Windows 桌面可通过 `\\tsclient\work` 访问。每条映射授予远程对该文件夹的**读写访问**；不会隐式共享主目录、系统盘或可移动盘。映射目录与经剪贴板发送文件是两个独立选择。

## 认证与证书策略

启用 TLS/NLA，TLS 最低 1.2；禁用旧式 RDP 安全及“忽略所有证书”。没有 pin 时，核验系统信任链、服务器用途、有效期和主机名/IP：Windows 用 CryptoAPI，macOS 用 SecTrust，Linux 用 OpenSSL 默认系统信任路径。Windows 链策略包括吊销检查。

宿主批准的 pin 是叶证书 DER 字节的精确 SHA-256（64 位十六进制）。**pin 替代 CA/名称/有效期验证**，只比较叶证书身份，不是在上述检查之上附加一道条件。叶证书变化会拒绝连接，需要重新明确批准。SSH 转发不会把预期服务器名称改成回环地址。密码通过 API 传递，不进入命令行参数；原生诊断可能包含端点信息。

## 构建与独立交付

保持仓库的 .NET 11、SDK 2.1.5 与 Avalonia 12.1.4。托管编译不要求暂存原生二进制，实际连接则需要。Linux/macOS 剪贴板文件测试也需要真实原生桥接库：它安全打开普通文件，避免 FIFO 阻塞并拒绝特殊文件。CI/release 在下载原生产物后于 Windows 执行托管测试；六个原生任务执行 CTest，不代表跨平台托管测试。原生构建需要 Git、PowerShell 7、CMake ≥3.25、C/C++17 编译器及 OpenSSL 开发文件；Linux 还需 pkg-config、zlib 和 UUID 开发文件。默认构建并行度 2，上限 4。工具链细节见 [插件构建 README](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/README.md) 和 [原生说明](https://github.com/VelaShellLabs/velashell-plugins/blob/main/plugins/VelaShell.Plugin.Rdp/native/README.md)。

`Build-RdpNative.ps1` 构建固定源码、运行离线 CTest、安装桥接库及非系统依赖/许可证，并测试安装后的共享库。因为要执行测试，RID 必须与原生 runner 匹配。默认输出为 `artifacts/rdp-native/<rid>/package/runtimes/<rid>/native`；把 `runtimes` 复制进 RDP 工程，或将 `-OutputDirectory` 指向该工程。生成的二进制不入 Git。Unix 库的符号链接别名会在上传产物前转换为普通文件，以便 Windows 打包。Windows 包含发现的 Visual C++ 运行库；macOS 调整 dylib 装载路径并作 ad-hoc 签名。系统库仍由操作系统提供。

CI 和发布复用同一个 `rdp-native.yml`，原生 runner 如下：

| RID | runner / 构建基线 |
| --- | --- |
| `win-x64` | `windows-2025` |
| `win-arm64` | `windows-11-arm` |
| `linux-x64` | `ubuntu-24.04` |
| `linux-arm64` | `ubuntu-24.04-arm` |
| `osx-x64` | `macos-15-intel` |
| `osx-arm64` | `macos-14` |

runner 可用性见 [GitHub 官方说明](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)。这是已配置的构建目标，不是已有发行资产的声明。Linux 需要兼容的 glibc/libstdc++；macOS 兼容性取决于构建部署目标。不由此推导 Alpine/musl 或更早系统已经受支持。

托管任务等待所有原生任务，通过 artifact 下载并合并六份产物到 RDP 工程，**再执行** `Bundle` / `PackVpx`。桥接库、FreeRDP/WinPR、OpenSSL、`LICENSE.FreeRDP`、`NOTICE` 或依赖许可证缺失都会导致打包失败。六种 RID 放在**同一个已签名 `.vpx`** 中，不分成六个插件。原生与托管发布构建检出相同 Release 标签，包括手动补跑。本地开发可以显式把 `RdpNativeRids` 缩为已暂存的单一 RID；正式流水线保持六种默认值。另见 [打包与签名](../templates/publishing.md)。

每个 RID 都携带 FreeRDP Apache-2.0 许可、notice 及随包依赖的许可证。插件遵循仓库许可证，不替换上游许可声明。

## 验证情况与剩余边界

还原依赖后，整个插件解决方案 882 项通过、196 项跳过、零失败。RDP 项目在 Release 下使用暂存的 Windows 原生库，65 项全部通过、无跳过，覆盖文本、PNG/DIB 像素往返、显式远端文件下载、本地文件请求、双向并发传输、输入、取消和释放。打包门禁已验证会拒绝缺失的必需 RID。本地未签名开发包 `plugins/VelaShell.Plugin.Rdp/bin/vpx/velashell.rdp-2.0.7.vpx` 仅包含 Windows/Linux x64：已通过 SDK 容器 API 读取全部 57 个条目，原生依赖与许可证齐全，没有混入 SDK/Avalonia 共享程序集或旧 IronRDP 程序集；它不是六平台签名发行包。中英文页面及索引的相对链接已在本地检查。

已有本地记录包括 Windows x64、WSL Debian Linux x64 原生构建、离线测试和安装后共享库 ABI 冒烟测试，覆盖缓冲、生命周期、通道/剪贴板处理、证书拒绝/pin 和回环取消。Debian 构建的 glibc 比 CI Ubuntu 基线新，它不是 Linux 发行资产。最新原生测试结果随实现记录。

macOS 和 ARM64 **尚未在本地执行**。新增六平台流水线实际通过后，才能称其产物可用。没有提供真实 RDP 端点：成功的实时 TLS/NLA、远程桌面绘制、真实剪贴板传输及映射文件夹互操作，仍需逐 OS/架构验收。离线测试和构建配置**不构成真实服务端或全平台认证**。
