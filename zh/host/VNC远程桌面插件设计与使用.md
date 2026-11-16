# VNC 远程桌面插件设计与使用

English: [VNC plugin: design and use](../../en/host/vnc-plugin-design-and-use.md)。

`velashell.vnc` 位于 [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Vnc),是进程内工作台插件,在 Windows、Linux、macOS 的 Avalonia 标签页中显示 VNC(RFB)远程桌面:Linux 桌面(TigerVNC、x11vnc 等)、macOS 屏幕共享、Windows 上的 VNC 服务,以及 QEMU/libvirt 等虚拟机控制台。引擎是**纯托管 C# 的 RFB 客户端**,不依赖原生 VNC 库,不捆绑原生二进制,不启动外部查看器窗口,全 RID 通用。

结构与 [RDP 插件](RDP远程桌面插件设计与使用.md)同一种分法:Plugin → Provider → Document → View → DesktopControl。

## 架构与范围

协议栈是独立的 C# 类库 `VelaShell.Vnc`(MIT 许可,随仓库发布):UI 无关、不依赖插件 SDK。行为依据是 RFC 6143(RFB 3.8)与社区协议登记(rfbproto)里下文列出的扩展。插件引用该库,库 DLL 随插件签名 `.vpx` 一起分发,不单独出 NuGet 包。分层:

- **Wire** —— 带缓冲的大端读写(`RfbReader` / `RfbWriter`);每条客户端消息一次写出。
- **Security** —— 托管 DES 与 VNC 认证、Apple 远程桌面、带证书钉扎的 VeNCrypt TLS(`VncTls`)。
- **Graphics** —— `VncFramebuffer`(越界检查、记录变化区域的 BGRA 帧缓冲)与 `RectangleDecoder`(全部编码;持续的 zlib 流在 `ZlibChannel` 里:ZRLE 一条、Tight 四条,服务器对每个矩形做 sync flush)。
- **Audio** —— QEMU 音频的播放器(`VncAudioPlayer`)与三平台 PCM 输出(与 RDP 插件同一份实现)。
- **FileTransfer** —— UltraVNC 文件传输(`UltraFileTransfer`)。
- **VncSession** —— 握手、接收循环、输入、剪贴板(`ExtendedClipboard`)、调整分辨率,以及音频与文件传输消息的分派。

| 方面 | 支持 |
| --- | --- |
| 协议版本 | RFB 3.3、3.7、3.8(Apple 的 3.889 按 3.8,3.4–3.6 按 3.3) |
| 安全类型 | None(1)、VNC 认证(2)、VeNCrypt 0.2 的 X.509 TLS 子类型(260 None / 261 VNC / 262 Plain)、Apple 远程桌面(30) |
| 编码 | Raw、CopyRect、RRE、Hextile、ZRLE、Tight(填充,copy / palette / gradient 过滤器,JPEG) |
| 伪编码 | ExtendedDesktopSize + SetDesktopSize、DesktopSize、Cursor、LastRect、DesktopName、QEMU Extended Key Event、QEMU Audio、Extended Clipboard、Continuous Updates + Fence、SupportedMessages(启用文件传输时)、JPEG 质量、压缩级别 |
| 剪贴板 | 仅双向文本:扩展剪贴板(Notify / Request / Provide)走 UTF-8,否则走 Latin-1 的 ServerCutText / ClientCutText;不支持文件剪贴板 |
| 音频 | QEMU Audio(QEMU / libvirt 控制台):S16 双声道 44.1 kHz,经 WaveOut / AudioQueue / ALSA 播放 |
| 文件传输 | UltraVNC 文件传输(UltraVNC Server、`x11vnc -ultrafilexfer`):浏览、上传、下载、新建文件夹、删除、重命名 |
| 传输 | 宿主网络代理(`INetworkApi`)、宿主管理的 SSH 跳板会话、30 秒 TCP keep-alive |

刻意不支持:匿名 TLS 的 VeNCrypt 子类型 257–259(匿名 Diffie-Hellman 套件,SChannel 与 .NET 的 OpenSSL/Apple TLS 绑定都不提供)、不带 TLS 的 VeNCrypt Plain(256,密码会明文传输)、RA2 / RealVNC 加密(5、6)、UltraVNC MS-Logon、TightVNC 的文件传输协议、整个文件夹的传输、音频输入(麦克风)、24 位以下色深、TightPNG、Open H.264(编码 50:常见服务器都不发送,只有 neatvnc 在编译时开启并配有 FFmpeg / V4L2 编码器时才有)。

库里的 `VncSession` 实现 `IVncSession` 契约(帧、输入、剪贴板、调整分辨率、显示暂停);插件提供桌面视图、输入队列与剪贴板桥接,并负责把宿主的连接请求映射成库的 `VncClientOptions`、把库抛出的异常翻译成宿主协议异常。宿主管理连接表单、凭据、证书批准、会话标签与可选 SSH 跳板。引擎在进程内运行,与宿主共享崩溃边界。相关契约见 [SDK 参考](../sdk/sdk-reference.md)、[开发指南](../templates/dev-guide.md) 与 [当前信任边界](../plugins/12-security-threat-model.md)。

服务器数据一律视为不可信:矩形越出桌面、桌面边长超过 16384、压缩载荷超过 64 MiB、解压后多于矩形容量、调色板下标越界、游程越出图块、光标大于 256×256,都会以协议错误结束会话,而不是分配大内存或越界写入。

## 安全类型

`security` 默认 `auto`,依次选择:VeNCrypt(服务器提供 X.509 子类型时)、Apple 远程桌面(填了用户名时)、VNC 认证、None,最后是未填用户名的 ARD(随后会要求输入用户名)。若 Auto 选了 VeNCrypt 而服务器只提供匿名 TLS 子类型,客户端会重新连接一次并选下一种类型 —— 同一条连接上无法更换安全类型。显式选择的类型从不回退到别的类型。

- **VNC 认证** 用密码前八个字符做 DES 密钥。DES 在托管代码里实现(`Security/Des.cs`,FIPS 46-3):OpenSSL 3 把单 DES 移进了多数 Linux 发行版不加载的 legacy provider,用平台密码库会让三个平台行为不一致。
- **VeNCrypt X.509**:证书由操作系统按目标主机名校验;自签证书走宿主的信任提示(`WorkspaceFeatures.CertificateTrust`),批准后的 SHA-256 指纹存进隐藏字段 `trustedThumbprint` 并从此钉扎。证书变化时再次提示,并说明是「变化」;已吊销的证书直接拒绝。经 SSH 隧道时,证书校验的是隧道的真实目标,而不是本地转发端点。
- **Apple 远程桌面**(macOS 屏幕共享):用服务器给的素数做 Diffie-Hellman,共享密钥的 MD5 作为 AES-128 密钥,加密块里放用户名与密码(UTF-8,各最多 63 字节)。
- 凭据由宿主的登录对话框索取:缺密码时在发送任何内容之前抛 `ProtocolAuthenticationException`,宿主随即重新询问。异常与日志永远不含凭据。

## 显示

- **显示尺寸**(`scaling`,「显示」页):默认「匹配窗口」—— 服务器回应了 ExtendedDesktopSize 时(例如 TigerVNC),经 SetDesktopSize 请求把远程桌面调整为标签页的设备像素大小,并沿用服务器给的屏幕 id;窗口停止变化 350 ms 后才发请求,8 px 以内的抖动忽略。不能调整的服务器按比例缩放。「缩放以适应」从不调整远端;「实际大小」1:1 绘制并带滚动条(比视图小时居中)。
- 桌面与视图相差几个设备像素以内时,按像素网格精确 1:1 绘制、不做滤波;只有真正不匹配时才高质量重采样。
- 帧在接收线程上解码进 BGRA 帧缓冲,缓冲记录变化矩形的并集;`CopyFrame` 只把这部分拷进锁定的 `WriteableBitmap`。帧投递用 Input 优先级。
- **连续更新**(`continuousUpdates`,高级,默认开):客户端声明 Continuous Updates(-313)与 Fence(-312)伪编码。服务器以一条 EndOfContinuousUpdates(150)确认支持 —— TigerVNC 只对同时支持 Fence 的客户端确认 —— 之后第一帧画面一到,客户端就对整个桌面发送 EnableContinuousUpdates(150),此后服务器在屏幕变化时主动推送,客户端不再逐帧请求;做法与 TigerVNC viewer 相同。服务器发来的 Fence 请求(248)立即原样回传负载:消息严格按序处理,因此保留 BlockBefore / BlockAfter,清掉 SyncNext 与 Request(与 TigerVNC viewer 的应答一致)。TigerVNC 服务器靠这些往返测量延迟、控制推送节奏,高延迟链路上比「每帧一问一答」流畅。桌面尺寸变化时按新尺寸重新开启;服务器自行结束连续更新时退回逐帧请求。不支持的服务器忽略这两个伪编码,仍是一帧解码完立即请求下一次增量更新。关掉此项也回到逐帧请求,仅用于排查问题。
- 后台标签页或最小化的窗口不再接收更新:连续更新时发送关闭消息,否则不再请求;重新显示时重新开启(或请求),服务器补发期间累积的变化(由服务器跟踪),不做整屏刷新。
- **画质**(`quality`):高(JPEG 8,默认)/ 中(6)/ 低(2)/ 无损(不用 JPEG)。JPEG 矩形借宿主的 Skia 经 Avalonia `Bitmap` 解码(`VncJpeg`),插件和库都不带图像编解码器。**首选编码**(`encoding`)与**压缩级别**(`compression`,0–9,默认 2)也在「显示」页。
- 服务器的光标(Cursor 伪编码)替换桌面上的本机指针;空光标即隐藏。

## 输入

- 键入的文本按每个字符的 keysym 发送(Latin-1 直接用,其余为 `0x01000000 + 码点`),因此无论两端键盘布局如何都按原样到达。没有文本的键(方向键、F 键、修饰键)与快捷键按物理键在美式布局上的 keysym 发送;服务器确认支持 QEMU Extended Key Event 后,同时附上 XT 扫描码(与布局无关,如同物理键盘)。
- AltGr(Windows,报告为 Ctrl+Alt)与 Option(macOS)键入字符时按文本处理:字符前后抬起再按下按住的 Ctrl/Alt。
- **macOS 上的 Command 键**(`macCommand`,高级):默认「按 Ctrl 发送」,在 Windows / Linux 桌面上 Cmd+C/V 即复制粘贴;远端是 Mac 时选「按 Super / ⌘ 发送」。
- 鼠标:RFB 按钮掩码(左/中/右),每个滚轮刻度是按钮 4–7 的一次按下与抬起,小数滚动量累计;指针移动会合并,慢速链路上不会积压。
- 单一有序输入队列(抬键不会越过它的按键);队列溢出时以「连接丢失」结束会话,而不是丢掉抬键。
- **远程按键悬浮球**(与 RDP 同一组件):Ctrl+Alt+Del、Alt+Tab、Alt+F4、键入剪贴板;外圈 Super、Win+R、Win+L、PrintScreen。「键入剪贴板」把本机剪贴板文本逐字键入(超过 100 字符或含换行需确认,超过 2000 字符拒绝)。
- **仅查看**(`viewOnly`)不发送键盘、指针或剪贴板数据,并隐藏远程按键悬浮球。

## 文本剪贴板

剪贴板桥接按顺序处理本地发布和远端写入，操作成功后才更新去重状态；焦点同步尚未完成时，粘贴前的重复发布会等待它完成。失败不会阻止随后重试相同文本。关闭视图会取消活动操作，并阻止排队操作继续开始；已经进入且不响应取消的宿主调用无法强制撤销。

扩展剪贴板 Caps 中的大小只限制未经请求的主动 Provide，不用于截断 Request 的响应。请求的文本完整发送，协议入口按 CRLF 规范化后的 UTF-8 字节数（含终止 NUL）检查 20 MiB 上限，超限在发布前拒绝。能力声明晚于首次焦点同步到达时，会在对端支持 Notify 的情况下重新宣告原始 Unicode 文本。这些源码修改及回归用例仍需构建和实际执行验证。

## 音频

- **播放远程声音**(`audio`,默认关,与 RDP 插件一致)。开启后客户端在 SetEncodings 里声明 QEMU Audio 伪编码(-259);服务器用一个编码为 -259 的矩形确认后(QEMU 在其中填桌面尺寸,没有意义),客户端先发 set-format(有符号 16 位、双声道、44.1 kHz),再发 enable。目前只有 QEMU / libvirt 控制台实现这个扩展,其他服务器忽略该伪编码。
- 样本按 QEMU 的默认字节序(小端)解读。PCM 输出与 RDP 插件同一份实现(Windows WaveOut、macOS AudioQueue、Linux ALSA,经 `LibraryImport` 调系统库),源码复制进库的 `Audio/` 目录 —— 插件各自分发,不能共享程序集。
- 接收线程只把数据交给播放器,从不阻塞:播放器攒成 40 ms 一段,输出端最多排 8 段(约 320 ms),延迟有上限。网络送得比设备播得快(时钟漂移、卡顿后的突发)时丢弃最新的数据 —— QEMU 自己在背压时也丢音频。
- 输出设备出错只停音频:状态栏提示一次(悬停状态栏可再看),并发 disable 让服务器停止采集;桌面不受影响。音频缓冲不是整帧时同样只停音频(跳过这段数据,流保持同步);超过 1 MiB 按协议错误处理。
- 音频启用时状态栏显示「声音」。

## 文件传输

RFB 本身没有文件传输。这里实现的是 UltraVNC 文件传输(消息类型 7):UltraVNC Server 和以 `-ultrafilexfer` 启动的 x11vnc(LibVNC)支持;TigerVNC、QEMU、macOS 屏幕共享不支持此扩展。TigerVNC 的文件传输请使用独立的 SSH/SFTP 连接;此次修复没有新增自动 SFTP 后端。UltraVNC 文件面板与剪贴板是独立功能:当前仅支持双向文本剪贴板,不支持复制粘贴文件。

- **文件传输**(`fileTransfer`,默认关)。启用时,`VncSession` 在 SetEncodings 中加入 SupportedMessages(`-131071`,`0xFFFE0001`),读取服务器返回的两个 32 字节位图,分别表示客户端到服务器和服务器到客户端的消息支持情况。默认只有两个位图都声明支持类型 7,才发送 UltraVNC 访问请求。打开文件面板请求访问时,最多等待 2 秒获取能力声明;未声明或明确不支持时,抛出错误类型为 `Unsupported` 的 `VncFileTransferException`,在面板内显示错误,保留桌面连接。「仅查看」时不提供文件传输。
- **旧服务器兼容**(`allowUnadvertisedFileTransfer`,高级布尔选项,默认 `false`,仅在 `fileTransfer=true` 时显示)。仅对已确认支持 UltraVNC 文件传输、但不声明能力的旧服务器启用。此项允许在没有能力声明时发送访问请求,不能覆盖明确不支持的声明。误用仍可能使桌面断线:不认识类型 7 的服务器可能直接关闭连接。
- **权限**:以 version 2 的 AbortFileTransfer 请求(version 0 是两种服务器都拒绝的旧握手),服务器回 FileTransferAccess(1 允许、-1 拒绝)。UltraVNC 随后发送协议版本消息:版本 ≥ 3 时客户端发 SessionStart;≥ 4 时路径与文件名走 Unicode(请求里置 0x80 标志,目录项为 UTF-16),并采用服务器给的块大小。LibVNC 不发版本消息,块大小为 8 KiB —— 它把每个上传包解压进 8192 字节的缓冲。
- **路径**一律是 Windows 形式(`C:\dir\file`),以 UTF-8 发送。LibVNC 只报告一个驱动器 `C:`,并把 `C:\` 映射到 `/`,所以在 Linux 服务器上从根目录开始浏览。
- 协议没有请求 id,所以一次只运行一个操作,应答都路由给它;没有操作在运行时到达的消息直接丢弃;一个操作中途失败(超时、取消、出错)后,下一个操作先丢弃到达的消息,直到服务器安静 500 ms,免得上一个操作迟到的应答被当成自己的。每个应答等 30 秒;权限请求等 15 秒,超时按拒绝处理(UltraVNC 对仅查看的会话不应答)。取消只在两条消息之间生效,不会把写到一半的包截断(那会让服务器读乱整条连接)。
- 文件头大小为 -1 既可能是「打不开」,也可能是低 32 位恰好全 1 的大文件;后者的消息体带时间戳,客户端据此区分,并照常读取高位字。**下载**请求压缩包(服务器只在压缩后更小时才压缩),逐包写入本地文件,收到 EndOfFile 结束;收到的字节少于文件头声明的大小算失败(LibVNC 的文件头只有 32 位大小,所以多了不算错)。**上传**先发 Offer(`路径,MM/DD/YYYY HH:MM`,后跟 4 字节高位大小),等到 FileAcceptHeader 后按块发送,可压缩的块压缩后再发,最后发 EndOfFile。取消或本地文件出错时向服务器发 AbortFileTransfer。
- **新建文件夹、删除、重命名**走 Command / CommandReturn。UltraVNC 删除文件夹会连同其中内容一起删除,LibVNC 只能删除空文件夹;删除文件夹的确认文字写明了这一点。
- 服务器数据同样不可信:消息体超过 2 MiB、解压后的包超过 1 MiB、单个目录超过 10 万项、畸形的驱动器列表,都以协议错误结束会话。

**文件弹窗**:状态栏右侧的文件夹按钮打开居中覆盖式弹窗,默认 800 × 600 px。拖动标题栏可在桌面区域内移动,拖动四边或四角可调整大小;周围留 16 px 边距,通常最小为 400 × 280 px,视口更小时优先缩小以保证关闭按钮可达。同一会话内重新打开会保留调整后的大小和位置。弹窗不占用远程桌面的布局空间,打开、移动、调整大小、关闭、重新打开都保持画面区域大小,不会因此触发远程分辨率切换。几何取自宿主:28 px 弹窗标题栏、36 px 工具栏、26 px 路径条、28 px 行、24 px 状态条。遮罩阻止鼠标操作穿透到远程桌面,Tab 在弹窗内循环;Esc 优先取消内联确认或名称输入,否则关闭弹窗并将焦点交还桌面。关闭只隐藏弹窗,保留浏览状态与正在进行的传输;断线时也会收起弹窗。首次打开时请求权限并列出驱动器,只有一个驱动器(LibVNC)时直接进入。工具栏是上一级、刷新、新建文件夹、上传;路径条右侧是对选中项的下载、重命名、删除。双击或回车进入文件夹、下载文件,Backspace 返回上一级,Delete 删除。把本机文件拖到弹窗上即上传到当前文件夹。删除确认、覆盖确认与输入名称都在弹窗内联完成,不另开对话框。上传时遇到同名文件整批确认一次,并先删除旧文件再上传 —— 否则 UltraVNC 会当作断点续传。文件夹不能整体上传或下载。下载失败或取消时删除不完整的本地文件。本地文件经宿主窗口的系统选择器选取(插件在进程内运行,做法与 Docker 面板相同),插件只读写用户选中的文件;清单为此声明了 `fs.local.read` / `fs.local.write`。

## 连接生命周期

- 网络断开(重置、超时)时按退避自动重连(1、2、4、8、15、30 秒,共 20 次);密码或证书变化时立即停止。
- 服务器主动关闭连接(会话结束,或别处的非共享查看者接管)时**不**自动重连 —— 两边都关了「共享」时那会变成互相踢 —— 断线卡片提供「重新连接」。
- `shared`(默认开)是 ClientInit 的共享标志;关掉会要求服务器断开其他查看者。

## 使用

从与其他插件相同的 Release 安装签名 `.vpx`,新建 VNC 工作台连接:服务器(默认端口 5900,显示号 `:1` 即 5901)、密码,以及仅 VeNCrypt Plain 或 macOS 需要的用户名。服务器只监听 localhost 时(常见的 `x11vnc -localhost` / `vncserver -localhost`),选一个已保存的 SSH 会话做跳板;宿主建好隧道,插件连接它的本地端点。

## 验证

`tests/VelaShell.Plugin.Vnc.Tests` 让真实的 `VncSession` 对接进程内脚本化 RFB 服务器(`FakeVncServer`),由服务器一侧逐字节核对客户端发出的内容:各版本映射、None / VNC / ARD(假服务器解密凭据)/ VeNCrypt X.509 自签证书(先信任提示、再钉扎重连;TLS 内的 Plain 凭据;证书变化),Auto 对只有匿名 TLS 的 VeNCrypt 的回退;DES 对照 FIPS 46-3 已知答案(Windows 上另对照平台实现);每种编码经真实读取器与帧缓冲解码,含畸形数据拒绝;增量请求、keysym 与 QEMU 扫描码消息、两种剪贴板双向、SetDesktopSize 往返、暂停显示、正常关闭 / 重置 / 协议错误;连续更新与 Fence(声明与关闭、确认后对整个桌面开启且不再请求、无确认时逐帧请求、Fence 应答的标志与负载、尺寸变化后重新开启、隐藏时关闭与显示时重开、服务器自行结束后退回请求);QEMU 音频的确认、set-format 与 enable 的字节、PCM 按序到达输出、设备故障只停音频并发 disable、非整帧缓冲的拒绝、输出跟不上时丢弃而不阻塞;UltraVNC 文件传输对照 LibVNC 与 UltraVNC 两种服务器行为(权限、驱动器与目录列表、Unicode 名称与块大小、压缩与原始包的下载、分块压缩上传、拒绝、取消即中止、命令、超时、不支持的服务器断开连接);插件侧的清单与注册一致、五种语言齐全且占位符一致、设置解析、异常翻译、自动重连规则、headless 界面(含文件面板的浏览、上传覆盖确认、删除确认、下载、重命名校验与错误提示,以及音频状态与故障提示)、经 Skia 解码 JPEG,以及在可回收加载上下文里卸载。

**原版插件的真实服务器观察:**在 Ubuntu 24 的 TigerVNC 1.13.1 上,发送类型 7 后服务器报 `unknown message type 7`,桌面连接随即断开。这是原有兼容问题的证据,不代表修复已验证。**本次修复尚未构建、运行测试或进行真实客户端/服务器验证**:本次更新时工具不提供本地命令执行能力。上文列出的是测试范围,不是测试通过记录。

待回归检查包括:SupportedMessages 声明及两个 32 字节位图;等待 2 秒后仍未声明或明确不支持时不发送类型 7;文件面板报错且桌面保持连接;兼容选项只允许缺少声明的情况,不能覆盖明确否定。需要分别在默认设置、启用文件传输但关闭兼容选项的情况下复测 TigerVNC 场景。

投入使用前还需连接 TigerVNC(`Xvnc -SecurityTypes VncAuth,X509Vnc`)、x11vnc、一个 QEMU/libvirt 控制台和 macOS 屏幕共享,检查登录、各首选编码下的画面、匹配窗口调整、非美式布局下的键入,以及双向文本剪贴板。音频要在带音频后端的 QEMU 虚拟机上试听;文件传输要对 UltraVNC Server(Windows)和 `x11vnc -ultrafilexfer` 各做一遍浏览、上传、下载、覆盖、删除、重命名、中文文件名和取消。兼容选项需另在已确认支持文件传输、但不声明能力的旧 UltraVNC 服务器上验证。
