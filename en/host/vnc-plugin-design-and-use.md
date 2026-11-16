# VNC remote desktop plugin: design and use

中文：[VNC 远程桌面插件设计与使用](../../zh/host/VNC远程桌面插件设计与使用.md).

`velashell.vnc` lives in [velashell-plugins](https://github.com/VelaShellLabs/velashell-plugins/tree/main/plugins/VelaShell.Plugin.Vnc). It is an in-process workspace plugin that shows a VNC (RFB) remote desktop in an Avalonia tab on Windows, Linux and macOS: Linux desktops (TigerVNC, x11vnc and others), macOS Screen Sharing, VNC services on Windows, and virtual-machine consoles such as QEMU/libvirt. The engine is a **pure-managed C# RFB client**: no native VNC library, no bundled native binaries, no external viewer window, one package for every RID.

It is split the same way as the [RDP plugin](rdp-plugin-design-and-use.md): Plugin → Provider → Document → View → DesktopControl.

## Architecture and scope

The protocol stack is the standalone C# library `VelaShell.Vnc` (MIT, released with the repository): UI-agnostic, with no plugin-SDK dependency. Behaviour follows RFC 6143 (RFB 3.8) and, for the extensions listed below, the community protocol registry (rfbproto). The plugin references the library, whose DLL ships inside the plugin's signed `.vpx`; there is no separate NuGet package. Layers:

- **Wire** — buffered big-endian reader/writer (`RfbReader` / `RfbWriter`); every client message is one write.
- **Security** — managed DES and VNC authentication, Apple Remote Desktop, VeNCrypt TLS with certificate pinning (`VncTls`).
- **Graphics** — `VncFramebuffer` (bounds-checked BGRA framebuffer that tracks its changed region) and `RectangleDecoder` (all encodings; persistent zlib streams in `ZlibChannel`: one for ZRLE, four for Tight, each rectangle sync-flushed by the server).
- **VncSession** — handshake, receive loop, input, clipboard (`ExtendedClipboard`), resizing.

| Area | Supported |
| --- | --- |
| Protocol versions | RFB 3.3, 3.7, 3.8 (Apple's 3.889 as 3.8; 3.4–3.6 as 3.3) |
| Security | None (1), VNC authentication (2), VeNCrypt 0.2 X.509 TLS subtypes (260 None / 261 VNC / 262 Plain), Apple Remote Desktop (30) |
| Encodings | Raw, CopyRect, RRE, Hextile, ZRLE, Tight (fill; copy / palette / gradient filters; JPEG) |
| Pseudo-encodings | ExtendedDesktopSize + SetDesktopSize, DesktopSize, Cursor, LastRect, DesktopName, QEMU Extended Key Event, Extended Clipboard, JPEG quality, compression level |
| Clipboard | Text both ways: UTF-8 through the extended clipboard (Notify / Request / Provide), Latin-1 ServerCutText / ClientCutText otherwise |
| Transport | Host network proxy (`INetworkApi`), host-managed SSH jump session, TCP keep-alive 30 s |

Deliberately not supported: anonymous-TLS VeNCrypt subtypes 257–259 (anonymous Diffie-Hellman cipher suites, which neither SChannel nor .NET's OpenSSL/Apple TLS bindings offer), VeNCrypt Plain without TLS (256, the password would travel in the clear), RA2 / RealVNC encryption (5, 6), UltraVNC MS-Logon, file transfer, audio, colour depths below 24 bits, TightPNG, Continuous Updates and Fence.

The library's `VncSession` implements the `IVncSession` contract (frames, input, clipboard, resizing, display pause). The plugin supplies the desktop view, input queue and clipboard bridge, maps the host's connect request to the library's `VncClientOptions`, and translates the library's exceptions into the host's protocol exceptions. The host owns the connection form, credentials, certificate approval, session tabs and the optional SSH jump host. The engine runs in-process and shares the host's crash boundary. See the [SDK reference](../sdk/sdk-reference.md), the [development guide](../templates/dev-guide.md) and the [current trust boundary](../plugins/12-security-threat-model.md).

Server data is untrusted: rectangles outside the desktop, desktop sides over 16384, compressed payloads over 64 MiB, inflated data larger than the rectangle allows, palette indices out of range, runs past a tile and cursors over 256×256 end the session with a protocol error instead of allocating or writing out of bounds.

## Security choice

`security` = `auto` (default) picks, in order: VeNCrypt (when the server offers an X.509 subtype), Apple Remote Desktop (when a username is given), VNC authentication, None, then ARD without a username (which then asks for one). If Auto picked VeNCrypt and the server turns out to offer only anonymous-TLS subtypes, the client connects once more and takes the next type — the security type cannot be changed on the same connection. An explicit choice never falls back to another type.

- **VNC authentication** uses DES keyed with the password's first eight characters. DES is implemented in managed code (`Security/Des.cs`, FIPS 46-3): OpenSSL 3 moved single DES to its legacy provider, which most Linux distributions do not load, so the platform cipher would make the three OSes behave differently.
- **VeNCrypt X.509**: the OS validates the certificate for the target host name; a self-signed certificate goes through the host's trust prompt (`WorkspaceFeatures.CertificateTrust`), and the approved SHA-256 thumbprint is stored in the hidden `trustedThumbprint` field and pinned from then on. A changed certificate is offered again, worded as a change; a revoked one is refused outright. Behind an SSH tunnel the certificate is checked against the tunnel's real target, not the local forwarding endpoint.
- **Apple Remote Desktop** (macOS Screen Sharing): Diffie-Hellman with the server's prime, MD5 of the shared secret as the AES-128 key, username and password (UTF-8, at most 63 bytes each) in the encrypted block.
- Credentials come from the host's login dialog: a missing password raises `ProtocolAuthenticationException` before anything is sent, and the host asks again. Exceptions and logs never contain credentials.

## Display

- **Display size** (`scaling`, Display page): *Match window* (default) asks the server, when it answered ExtendedDesktopSize (for example TigerVNC), to resize its desktop to the tab in device pixels through SetDesktopSize, keeping the server's screen id; requests go out 350 ms after the window stops changing and ignore jitter under 8 px. Servers that cannot resize are scaled to fit. *Scale to fit* never resizes the remote; *Actual size* draws 1:1 with scroll bars (centred when smaller than the view).
- A desktop within a few device pixels of the view is drawn exactly 1:1 on the pixel grid with no filtering; only a real mismatch is resampled (high quality).
- Frames are decoded on the receive thread into a BGRA framebuffer that keeps the union of changed rectangles; `CopyFrame` copies only that into the locked `WriteableBitmap`. The next incremental update is requested as soon as one has been decoded. A hidden tab or minimized window stops requesting updates; showing it again requests what changed meanwhile (the server tracks it). Frame posts use Input priority.
- **Picture quality** (`quality`): High (JPEG 8, default) / Medium (6) / Low (2) / Lossless (no JPEG). JPEG rectangles are decoded by the host's Skia through Avalonia's `Bitmap` (`VncJpeg`), so neither the plugin nor the library ships an image codec. **Preferred encoding** (`encoding`) and **Compression level** (`compression`, 0–9, default 2) are on the Display page too.
- The server's cursor (Cursor pseudo-encoding) replaces the local pointer over the desktop; an empty cursor hides it.

## Input

- Typed text goes as the keysym of each character (Latin-1 directly, everything else `0x01000000 + code point`), so it arrives as typed whatever the two keyboard layouts are. Keys without text (arrows, F-keys, modifiers) and shortcuts go as the US-layout keysym of the physical key; once the server confirmed the QEMU Extended Key Event, the XT scan code goes with it (layout-independent, like a physical keyboard).
- AltGr (Windows, reported as Ctrl+Alt) and Option (macOS) typing a character is treated as text: the held Ctrl/Alt is lifted around the character and pressed again.
- **Command key on macOS** (`macCommand`, advanced): *Send as Ctrl* (default) so Cmd+C/V copy and paste on a Windows or Linux desktop; *Send as Super / ⌘* for a Mac server.
- Mouse: RFB button mask (left/middle/right), each wheel notch as a press and release of buttons 4–7, fractional wheel deltas carried over; pointer moves are coalesced so a slow link never builds a backlog.
- One ordered input queue (a key-up never overtakes its key-down); an overflowing queue ends the session as a lost connection rather than dropping a key-up.
- **Remote-keys floating button** (the same component as RDP's): Ctrl+Alt+Del, Alt+Tab, Alt+F4, Type clipboard; outer ring Super, Win+R, Win+L, PrintScreen. *Type clipboard* types the local clipboard text as keystrokes (confirmation above 100 characters or with line breaks, refused above 2000).
- **View only** (`viewOnly`) sends no keyboard, pointer or clipboard data and hides the remote-keys button.

## Connection lifetime

- A dropped network connection (reset, timeout) reconnects automatically with back-off (1, 2, 4, 8, 15, 30 s; 20 attempts), stopping at once when the password or certificate changed.
- A server that closes the connection itself (session ended, or a non-shared viewer elsewhere took over) is **not** reconnected automatically — with `shared` off on both sides that would start a tug of war — and the disconnect card offers Reconnect.
- `shared` (default on) is the ClientInit shared flag; turning it off asks the server to disconnect other viewers.

## Use

Install the signed `.vpx` from the same release as the other plugins and create a VNC workspace connection: server (default port 5900 — display `:1` is 5901), password, and a username only for VeNCrypt Plain or macOS. For a server bound to localhost (the usual `x11vnc -localhost` / `vncserver -localhost` setup), pick a saved SSH session as the jump host; the host builds the tunnel and the plugin connects to its local endpoint.

## Verification

`tests/VelaShell.Plugin.Vnc.Tests` runs the real `VncSession` against a scripted in-process RFB server (`FakeVncServer`) that checks the client's bytes from the server side: every version mapping; None / VNC / ARD (the fake server decrypts the credentials) / VeNCrypt X.509 with a self-signed certificate (trust prompt, then a pinned reconnect; Plain credentials inside TLS; a changed certificate); Auto's fallback from anonymous-only VeNCrypt; DES against the FIPS 46-3 known answer (and, on Windows, the platform cipher); every encoding through the real reader and framebuffer, including malformed-data rejection; incremental requests, keysym vs. QEMU scan-code messages, both clipboards in both directions, the SetDesktopSize round trip, paused display, clean close vs. reset vs. protocol error; and on the plugin side manifest vs. registration, all five languages complete with matching placeholders, settings parsing, exception translation, auto-reconnect rules, the headless UI, JPEG decoding through Skia, and unloading in a collectible load context.

**Not yet verified against real servers.** Before relying on it, connect to at least TigerVNC (`Xvnc -SecurityTypes VncAuth,X509Vnc`), x11vnc, a QEMU/libvirt console and macOS Screen Sharing, and check login, rendering with each preferred encoding, Match window resizing, typing on a non-US layout, and the clipboard both ways.
