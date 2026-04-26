---
title: "Double Tunnel"
date: 2026-04-25 21:30:18 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, tunneling, ssh, socks, double-tunnel]
description: "HTB CPTS note về kỹ thuật double tunnel khi pivot qua nhiều lớp mạng."
pin: false
---
title: RDP and SOCKS Tunneling with SocksOverRDP
tags:
  - CPTS
  - Pivoting
  - Tunneling
  - RDP
  - SOCKS
  - Windows
  - Lateral-Movement
  - Proxy
aliases:
  - SocksOverRDP
  - RDP SOCKS Tunnel
  - Double Tunnel
---

---
# RDP and SOCKS Tunneling with SocksOverRDP

## Mục tiêu

Sử dụng **RDP Dynamic Virtual Channel (DVC)** để tạo **SOCKS proxy tunnel** trong môi trường Windows, đặc biệt khi không thể dùng SSH để pivot.

Kỹ thuật này hữu ích khi:

- Chỉ có quyền truy cập RDP vào Windows host.
- Không có SSH để dùng `ssh -D`.
- Cần pivot từ một Windows foothold vào subnet nội bộ sâu hơn.
- Cần ép traffic của Windows app như `mstsc.exe` đi qua proxy.

---
## Lab topology mẫu

```text
Kali Attacker
  |
  | RDP / xfreerdp
  v
Windows Foothold
  - Có SocksOverRDP-Plugin.dll
  - Có Proxifier
  - Có SOCKS listener 127.0.0.1:1080
  |
  | mstsc.exe RDP
  v
Pivot Host 172.16.5.19
  - Chạy SocksOverRDP-Server.exe
  |
  v
Internal Target 172.16.6.155
  - Ví dụ: RDP 3389
```


---

## Ý tưởng chính

> **Bản chất**
> **SocksOverRDP biến một phiên RDP thành đường hầm SOCKS.**
> Traffic từ app Windows được đưa vào `127.0.0.1:1080`, sau đó được đóng gói qua **RDP Dynamic Virtual Channel** để đi tới host nội bộ.

Luồng tổng quát:

```text
Windows App
  -> Proxifier
  -> SOCKS 127.0.0.1:1080
  -> RDP Dynamic Virtual Channel
  -> Pivot Host
  -> Internal Target
```

---
## Dynamic Virtual Channel là gì?

**Dynamic Virtual Channel (DVC)** là cơ chế trong RDP dùng để truyền dữ liệu phụ trợ ngoài màn hình chính, ví dụ:

- Clipboard copy/paste
- Audio redirection
- Device redirection
- Printer redirection
- File/drive sharing

SocksOverRDP lợi dụng DVC để truyền traffic tùy ý qua phiên RDP.

---
## Double Tunnel là gì?

> **Double Tunnel**
> Trong bài này, “double tunnel” nghĩa là traffic đi qua nhiều lớp tunnel lồng nhau:
>
> 1. Lớp ngoài: phiên RDP.
> 2. Lớp trong: SOCKS proxy chạy qua RDP Dynamic Virtual Channel.

Sơ đồ:

```text
Kali Attacker
   |
   | RDP / xfreerdp
   v
Windows Foothold
   |
   | SOCKS 127.0.0.1:1080
   |
   | Traffic được đóng gói vào RDP DVC
   v
Pivot Host 172.16.5.19
   |
   | SocksOverRDP-Server.exe forward tiếp
   v
Internal Target 172.16.6.155
```

Nói ngắn gọn:

```text
Tunnel 1: Kali -> RDP -> Windows Foothold
Tunnel 2: Windows Foothold -> RDP DVC SOCKS Tunnel -> Pivot Host
Final: Pivot Host -> Internal Target
```

Điểm quan trọng:

> Internal target sẽ thấy kết nối đến từ **pivot host**, không phải trực tiếp từ Kali.

---
## Thành phần cần chuẩn bị

| Thành phần | Vai trò |
|---|---|
| `SocksOverRDP-Plugin.dll` | Plugin phía RDP client, tạo SOCKS listener |
| `SocksOverRDP-Server.exe` | Server component chạy trên RDP target |
| `regsvr32.exe` | Đăng ký DLL plugin vào Windows |
| `mstsc.exe` | Windows Remote Desktop Client |
| `Proxifier` | Ép ứng dụng Windows đi qua SOCKS proxy |
| `127.0.0.1:1080` | Local SOCKS proxy port |

> **> `SocksOverRDP-Plugin.dll` nằm trên **Windows foothold**.**
>
> `SocksOverRDP-Server.exe` nằm trên **pivot host `172.16.5.19`**.

---
## Quy trình triển khai

### 1. Copy SocksOverRDP vào Windows foothold

Từ attack host, copy bộ SocksOverRDP sang máy Windows đã có quyền truy cập RDP.

Ví dụ thư mục:

```text
C:\Tools\SocksOverRDP
```

---
### 2. Register SocksOverRDP Plugin

Trên Windows foothold, chạy CMD/PowerShell bằng quyền Administrator:

```cmd
cd /d C:\Tools\SocksOverRDP
C:\Windows\System32\regsvr32.exe "C:\Tools\SocksOverRDP\SocksOverRDP-Plugin.dll"
```

Ý nghĩa:

- `regsvr32.exe`: đăng ký DLL COM trên Windows.
- `SocksOverRDP-Plugin.dll`: plugin tích hợp vào RDP client.
- Sau khi register thành công, plugin có thể tạo RDP virtual channel.

Kết quả mong đợi:

```text
DllRegisterServer in SocksOverRDP-Plugin.dll succeeded.
```

---
### 3. RDP từ foothold sang pivot host

Từ Windows foothold, mở `mstsc.exe` và RDP tới pivot host:

```text
172.16.5.19
```

Khi kết nối thành công, plugin sẽ báo SOCKS listener được bật tại:

```text
127.0.0.1:1080
```

> **> `127.0.0.1:1080` nằm trên **Windows foothold**, không phải trên Kali và cũng không phải trên pivot host.**

---
### 4. Copy SocksOverRDP-Server.exe sang pivot host

Sau khi đã RDP vào `172.16.5.19`, copy `SocksOverRDP-Server.exe` sang máy này.

Có thể dùng:

- RDP clipboard copy/paste
- Drive redirection qua `\\tsclient\C`
- File share nội bộ nếu có

Ví dụ copy từ redirected drive:

```cmd
mkdir C:\Tools\SocksOverRDP
copy "\\tsclient\C\Tools\SocksOverRDP\SocksOverRDP-Server.exe" "C:\Tools\SocksOverRDP\SocksOverRDP-Server.exe"
```

---
### 5. Chạy SocksOverRDP Server trên pivot host

Trên pivot host `172.16.5.19`, mở CMD/PowerShell bằng quyền Administrator:

```cmd
cd /d C:\Tools\SocksOverRDP
SocksOverRDP-Server.exe
```

Thường **không cần option gì thêm**.

Khi server chạy đúng, sẽ có output kiểu:

```text
Socks Over RDP
Channel opened
```

> **> Không đóng cửa sổ `SocksOverRDP-Server.exe`.**
> Nếu đóng cửa sổ này thì tunnel sẽ chết.

---
### 6. Kiểm tra SOCKS listener trên foothold

Quay lại Windows foothold, chạy:

```cmd
netstat -antb | findstr 1080
```

Kết quả mong đợi:

```text
TCP    127.0.0.1:1080    0.0.0.0:0    LISTENING
```

Nếu thấy port `1080` listening, SOCKS tunnel đã hoạt động.

---
## Cấu hình Proxifier

Sau khi SocksOverRDP tunnel hoạt động, trên **Windows foothold** sẽ có SOCKS listener tại:

```text
127.0.0.1:1080
```

Lúc này cần cấu hình **Proxifier** để ép traffic của các ứng dụng Windows như `mstsc.exe` đi qua SOCKS proxy này.

> **> Proxifier phải chạy trên **Windows foothold**, tức máy đã register `SocksOverRDP-Plugin.dll`.**
> Không cấu hình Proxifier trên Kali và cũng không cấu hình trên pivot host `172.16.5.19`.

---
### 1. Thêm SOCKS proxy vào Proxifier

Mở **Proxifier** trên Windows foothold:

```text
Profile -> Proxy Servers -> Add
```

Cấu hình:

| Trường | Giá trị |
|---|---|
| Address | `127.0.0.1` |
| Port | `1080` |
| Protocol | `SOCKS Version 5` |
| Authentication | Không dùng, để trống |

Sau đó bấm:

```text
Check
```

> **> Nếu nút `Check` fail nhưng `netstat` vẫn thấy `127.0.0.1:1080 LISTENING`, vẫn có thể thử dùng `mstsc.exe` để test thực tế. Một số SOCKS tunnel có thể không phản hồi tốt với test mặc định của Proxifier.**

---
### 2. Tạo Proxification Rule cho mstsc.exe

Vào:

```text
Profile -> Proxification Rules
```

Bấm **Add** và tạo rule mới:

| Trường | Giá trị |
|---|---|
| Name | `RDP through SocksOverRDP` |
| Applications | `mstsc.exe` |
| Target hosts | `172.16.6.155` |
| Target ports | `3389` |
| Action | Proxy `127.0.0.1:1080 SOCKS5` |

Nếu chỉ muốn route đúng target cụ thể:

```text
Applications: mstsc.exe
Target hosts: 172.16.6.155
Target ports: 3389
Action: SOCKS5 127.0.0.1:1080
```

Nếu muốn route tất cả RDP traffic qua tunnel:

```text
Applications: mstsc.exe
Target hosts: Any
Target ports: 3389
Action: SOCKS5 127.0.0.1:1080
```

> **> Rule càng cụ thể càng tốt. Không nên cho toàn bộ traffic Windows đi qua SOCKS nếu chưa cần, vì dễ gây lỗi hoặc làm session chậm.**

---
### 3. Đảm bảo rule order đúng

Trong **Proxification Rules**, rule của `mstsc.exe` nên nằm **trên Default rule**.

Thứ tự khuyến nghị:

```text
1. RDP through SocksOverRDP -> Proxy SOCKS5 127.0.0.1:1080
2. Localhost -> Direct
3. Default -> Direct
```

Nếu có rule `Default -> Proxy`, Proxifier có thể ép quá nhiều traffic qua SOCKS, gây chậm hoặc lỗi.

---
### 4. Bật DNS qua proxy nếu dùng hostname

Nếu truy cập bằng IP như:

```text
172.16.6.155
```

thì không cần DNS.

Nếu truy cập bằng hostname nội bộ như:

```text
DC01.internal.local
```

thì vào:

```text
Profile -> Name Resolution
```

Chọn:

```text
Resolve hostnames through proxy
```

> **> Chỉ bật DNS through proxy khi thật sự cần resolve hostname nội bộ. Nếu dùng IP trực tiếp thì có thể bỏ qua.**

---
## Test RDP tới internal target

Sau khi cấu hình xong, mở `mstsc.exe` trên Windows foothold:

```cmd
mstsc.exe
```

Nhập target sâu hơn:

```text
172.16.6.155
```

Nếu Proxifier hoạt động, trong cửa sổ log của Proxifier sẽ thấy connection kiểu:

```text
mstsc.exe -> 172.16.6.155:3389 -> 127.0.0.1:1080
```

Luồng thực tế:

```text
mstsc.exe
  -> Proxifier
  -> SOCKS5 127.0.0.1:1080
  -> SocksOverRDP Plugin
  -> RDP Dynamic Virtual Channel
  -> Pivot Host 172.16.5.19
  -> Internal Target 172.16.6.155:3389
```

---
## So sánh với SSH Dynamic Port Forwarding

| SSH Dynamic Port Forwarding | SocksOverRDP |
|---|---|
| Dùng SSH | Dùng RDP |
| Phù hợp Linux/Kali | Phù hợp Windows-only environment |
| Lệnh kiểu `ssh -D` | Dùng DLL plugin + RDP DVC |
| Proxychains thường dùng ở Kali | Proxifier thường dùng trên Windows |
| Cần SSH access | Chỉ cần RDP access |
| Tạo SOCKS proxy local | Cũng tạo SOCKS proxy local |

Ví dụ SSH dynamic port forwarding:

```bash
ssh -D 1080 user@pivot
```

Ý tưởng tương đương trong SocksOverRDP:

```text
RDP Session + Dynamic Virtual Channel -> SOCKS 127.0.0.1:1080
```

---
## RDP Performance Considerations

Khi RDP lồng RDP hoặc tunnel nhiều lớp, session có thể chậm.

Nguyên nhân:

```text
RDP GUI traffic
+ Clipboard/audio/device channel
+ SOCKS traffic
+ Nested RDP traffic
```

Nên chỉnh trong `mstsc.exe`:

```text
Show Options -> Experience -> Performance -> Modem / Low bandwidth
```

Tắt bớt:

- Desktop background
- Font smoothing
- Animation
- Visual styles
- Audio redirection nếu không cần
- Clipboard nếu không cần

---

## Khi nào dùng kỹ thuật này?

Dùng khi:

- Đã có RDP foothold trên Windows.
- Không có SSH để pivot.
- Cần reach subnet nội bộ sâu hơn.
- Cần dùng Windows-native tools như `mstsc.exe`.
- Lab hoặc assessment bị giới hạn trong Windows network.

Không tối ưu khi:

- Có SSH ổn định.
- Có VPN hoặc route trực tiếp.
- RDP session quá lag.
- EDR kiểm soát chặt DLL registration hoặc RDP plugin.

---
## Detection Note

### Dấu hiệu trên host

| Dấu hiệu | Ý nghĩa |
|---|---|
| `regsvr32.exe` load DLL lạ | Có thể có plugin/tunnel DLL |
| DLL lạ được register | Khả năng abuse COM/DLL registration |
| Process listen `127.0.0.1:1080` | Local SOCKS proxy |
| `mstsc.exe` tạo kết nối bất thường | RDP lateral movement |
| Process lạ chạy với Admin trên pivot host | Có thể là tunnel server |

---
### Dấu hiệu trên network

| Dấu hiệu | Ý nghĩa |
|---|---|
| RDP session kéo dài bất thường | Possible tunnel over RDP |
| RDP traffic volume cao bất thường | Có thể đang carry thêm SOCKS traffic |
| Pivot host kết nối nhiều host nội bộ | Lateral movement/pivoting |
| Một host RDP tới nhiều subnet lạ | Internal discovery/pivot behavior |

---
## MITRE ATT&CK Mapping

| Technique | ID | Ý nghĩa |
|---|---|---|
| Remote Services: RDP | `T1021.001` | Dùng RDP để truy cập/lateral movement |
| Proxy | `T1090` | Dùng proxy để chuyển tiếp traffic |
| Protocol Tunneling | `T1572` | Tunnel traffic qua protocol hợp lệ |
| Regsvr32 | `T1218.010` | Dùng signed binary để load/register DLL |

---
