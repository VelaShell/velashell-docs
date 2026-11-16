# VNC 远程桌面插件设计与使用

English: [VNC plugin: design and use](../../en/host/vnc-plugin-design-and-use.md)。

`velashell.vnc` 位于 [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Vnc),是进程内工作台插件,在 Windows、Linux、macOS 的 Avalonia 标签页中显示 VNC(RFB)远程桌面:Linux 桌面(TigerVNC、x11vnc 等)、macOS 屏幕共享、Windows 上的 VNC 服务,以及 QEMU/libvirt 等虚拟机控制台。引擎是**纯托管 C# 的 RFB 客户端**,不依赖原生 VNC 库,不捆绑原生二进制,不启动外部查看器窗口,全 RID 通用。

结构与 [RDP 插件](RDP远程桌面插件设计与使用.md)同一种分法:Plugin → Provider → Document → View → DesktopControl。

## 架构与范围

协议栈是独立的 C# 类库 `VelaShell.Vnc`(MIT 许可,随仓库发布):UI 无关、不依赖插件 SDK。行为依据是 RFC 6143(RFB 3.8)与社区协议登记(rfbproto)里下文列出的扩展。插件引用该库,库 DLL 随插件签名 `.vpx` 一起分发,不单独出 NuGet 包。分层:

- **Wire** —— 带缓冲的大端读写(`RfbReader` / `RfbWriter`);每条客户端消息一次写出。
- **Security** —— 托管 DES 与 VNC 认证、Apple 远程桌面、带证书钉扎的 VeNCrypt TLS(`VncTls`)。
- **Graphics** —— `VncFramebuffer`(越界检查、记录变化区域的 BGRA 帧缓冲)与 `RectangleDecoder`(全部编码;持续的 zlib 流在 `ZlibChannel` 里:ZRLE 一条、Tight 四条,服务器对每个矩形做 sync flush)。
- **VncSession** —— 握手、接收循环、输入、剪贴板(`ExtendedClipboard`)、调整分辨率。

| 方面 | 支持 |
| --- | --- |
| 协议版本 | RFB 3.3、3.7、3.8(Apple 的 3.889 按 3.8,3.4–3.6 按 3.3) |
| 安全类型 | None(1)、VNC 认证(2)、VeNCrypt 0.2 的 X.509 TLS 子类型(260 None / 261 VNC / 262 Plain)、Apple 远程桌面(30) |
| 编码 | Raw、CopyRect、RRE、Hextile、ZRLE、Tight(填充,copy / palette / gradient 过滤器,JPEG) |
| 伪编码 | ExtendedDesktopSize + SetDesktopSize、DesktopSize、Cursor、LastRect、DesktopName、QEMU Extended Key Event、Extended Clipboard、JPEG 质量、压缩级别 |
| 剪贴板 | 双向文本:扩展剪贴板(Notify / Request / Provide)走 UTF-8,否则走 Latin-1 的 ServerCutText / ClientCutText |
| 传输 | 宿主网络代理(`INetworkApi`)、宿主管理的 SSH 跳板会话、30 秒 TCP keep-alive |

刻意不支持:匿名 TLS 的 VeNCrypt 子类型 257–259(匿名 Diffie-Hellman 套件,SChannel 与 .NET 的 OpenSSL/Apple TLS 绑定都不提供)、不带 TLS 的 VeNCrypt Plain(256,密码会明文传输)、RA2 / RealVNC 加密(5、6)、UltraVNC MS-Logon、文件传输、音频、24 位以下色深、TightPNG、Continuous Updates 与 Fence。

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
- 帧在接收线程上解码进 BGRA 帧缓冲,缓冲记录变化矩形的并集;`CopyFrame` 只把这部分拷进锁定的 `WriteableBitmap`。一帧解码完立即请求下一次增量更新。后台标签页或最小化的窗口不再请求更新;重新显示时请求期间累积的变化(由服务器跟踪)。帧投递用 Input 优先级。
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

## 连接生命周期

- 网络断开(重置、超时)时按退避自动重连(1、2、4、8、15、30 秒,共 20 次);密码或证书变化时立即停止。
- 服务器主动关闭连接(会话结束,或别处的非共享查看者接管)时**不**自动重连 —— 两边都关了「共享」时那会变成互相踢 —— 断线卡片提供「重新连接」。
- `shared`(默认开)是 ClientInit 的共享标志;关掉会要求服务器断开其他查看者。

## 使用

从与其他插件相同的 Release 安装签名 `.vpx`,新建 VNC 工作台连接:服务器(默认端口 5900,显示号 `:1` 即 5901)、密码,以及仅 VeNCrypt Plain 或 macOS 需要的用户名。服务器只监听 localhost 时(常见的 `x11vnc -localhost` / `vncserver -localhost`),选一个已保存的 SSH 会话做跳板;宿主建好隧道,插件连接它的本地端点。

## 验证

`tests/VelaShell.Plugin.Vnc.Tests` 让真实的 `VncSession` 对接进程内脚本化 RFB 服务器(`FakeVncServer`),由服务器一侧逐字节核对客户端发出的内容:各版本映射、None / VNC / ARD(假服务器解密凭据)/ VeNCrypt X.509 自签证书(先信任提示、再钉扎重连;TLS 内的 Plain 凭据;证书变化),Auto 对只有匿名 TLS 的 VeNCrypt 的回退;DES 对照 FIPS 46-3 已知答案(Windows 上另对照平台实现);每种编码经真实读取器与帧缓冲解码,含畸形数据拒绝;增量请求、keysym 与 QEMU 扫描码消息、两种剪贴板双向、SetDesktopSize 往返、暂停显示、正常关闭 / 重置 / 协议错误;插件侧的清单与注册一致、五种语言齐全且占位符一致、设置解析、异常翻译、自动重连规则、headless 界面、经 Skia 解码 JPEG,以及在可回收加载上下文里卸载。

**尚未对接真实服务器验证。** 投入使用前至少连接 TigerVNC(`Xvnc -SecurityTypes VncAuth,X509Vnc`)、x11vnc、一个 QEMU/libvirt 控制台和 macOS 屏幕共享,检查登录、各首选编码下的画面、匹配窗口调整、非美式布局下的键入,以及双向剪贴板。
