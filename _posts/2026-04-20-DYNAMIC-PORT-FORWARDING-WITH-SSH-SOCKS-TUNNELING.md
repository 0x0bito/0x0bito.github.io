---
title: "Dynamic Port Forwarding with SSH and SOCKS Tunneling"
date: 2026-04-20 17:30:34 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, ssh, socks, proxychains, port-forwarding]
description: "HTB CPTS note về SSH dynamic port forwarding, SOCKS tunneling và proxychains."
pin: false
---
## tags: [pivoting, tunneling, ssh, socks, proxychains, htb] module: "Pivoting, Tunneling & Port Forwarding"
---
---
# SSH Port Forwarding & SOCKS Tunneling

## Table of Contents

- Bản chất
- 1. SSH Local Port Forwarding
    - Bản chất kỹ thuật
    - Luồng từ góc nhìn Attacker
    - Lệnh
    - Xác minh
- 2. Dynamic Port Forwarding + SOCKS Tunneling
    - Khi nào dùng
    - Bản chất SOCKS
    - Luồng thực tế
    - Các bước thực hiện
    - Hạn chế quan trọng
- 3. Phát hiện cơ hội Pivot
- Quick Reference
- Ghi chú thực tế

---
## Bản chất

**Port Forwarding** là kỹ thuật chuyển hướng kết nối từ một cổng này sang cổng khác. Trong ngữ cảnh tấn công mạng, nó được dùng để **truy cập các dịch vụ nằm sau tường lửa** hoặc trong **subnet mà attacker không có route trực tiếp**.

SSH là phương tiện phổ biến nhất để thực hiện port forwarding vì nó mã hóa traffic và thường được phép qua firewall.

---
## 1. SSH Local Port Forwarding

### Bản chất kỹ thuật

> "Tôi muốn dịch vụ đang chạy trên máy **victim** xuất hiện như thể đang chạy trên máy **local** của tôi."

Khi attacker đã SSH được vào pivot host, có thể có dịch vụ trong mạng nội bộ mà attacker chưa truy cập được trực tiếp. Local port forwarding tạo một "đường ống" qua SSH để forward traffic từ cổng local của attacker đến dịch vụ đích.

```
Attack Host :1234  ──SSH tunnel──►  Pivot Host  ──►  localhost
```

### Lệnh

```bash
# Forward 1 cổng
ssh -L 1234:localhost:3306 ubuntu@<PIVOT_IP>

# Forward nhiều cổng cùng lúc
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@<PIVOT_IP>
```

> **Giải thích cú pháp:** `-L <local_port>:<target_host>:<target_port>`
>
> - `local_port` = cổng mở trên máy attacker
> - `target_host` = host đích **tính từ góc nhìn của pivot** (có thể là `localhost` hoặc IP trong mạng nội bộ)
> - `target_port` = cổng dịch vụ đích

### Xác minh

```bash
# Kiểm tra cổng đã được mở chưa
netstat -antp | grep 1234

# Scan bằng Nmap để xác nhận dịch vụ
nmap -v -sV -p1234 localhost
```

---
## 2. Dynamic Port Forwarding + SOCKS Tunneling

### Khi nào dùng?

Khi attacker **không biết trước cổng/dịch vụ cụ thể** cần forward. Thay vì forward từng cổng riêng lẻ, tạo một **SOCKS proxy** để route toàn bộ traffic tùy ý qua pivot host vào subnet nội bộ.

### Bản chất SOCKS

**SOCKS (Socket Secure)** là giao thức proxy hoạt động ở tầng transport. Khác với HTTP proxy (chỉ hiểu HTTP), SOCKS có thể proxy **bất kỳ TCP traffic nào**.

|Tính năng|SOCKS4|SOCKS5|
|---|---|---|
|Authentication|✗|✓|
|UDP support|✗|✓|

Trong SSH dynamic port forwarding, SSH client tự đóng vai trò là **SOCKS server**:

```
[Tool] → proxychains → localhost:9050 (SOCKS) → SSH tunnel → Pivot Host → Internal Network
```

### Luồng thực tế

```
Attack Host                    Pivot Host              Internal Network
   │                               │                        │
   │  ssh -D 9050                  │                        │
   ├──────────────────────────────►│                        │
   │                               │                        │
   │  proxychains nmap 172.16.5.x  │                        │
   │──►localhost:9050              │                        │
   │      │ (SOCKS listener)       │                        │
   │      └──SSH tunnel───────────►│──────►172.16.5.x:port  │
   │                               │◄──────response─────────│
   │◄──────────────────────────────│                        │
```

### Các bước thực hiện

**Bước 1: Mở SOCKS proxy qua SSH**

```bash
ssh -D 7807 ubuntu@<PIVOT_IP>
```

> `-D 7807` = SSH client mở SOCKS listener tại `localhost:9050`

**Bước 2: Cấu hình proxychains**

```bash
tail -4 /etc/proxychains.conf
# Đảm bảo có dòng:
# socks4  127.0.0.1 9050
```

**Bước 3: Chạy tool qua proxychains**

```bash
# Ping sweep subnet nội bộ
proxychains nmap -v -sn 172.16.5.1-200

# Full scan một host cụ thể (-Pn vì ICMP không qua SOCKS)
proxychains nmap -v -Pn -sT 172.16.5.19

# Mở Metasploit qua proxy
proxychains msfconsole

# RDP vào host nội bộ
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

### Hạn chế quan trọng

> **Chỉ dùng được Full TCP Connect Scan proxychains không thể xử lý **partial packets**. Với Nmap phải dùng `-sT` (full connect scan), không dùng được `-sS` (SYN/half-open scan).**

> **Windows host không trả lời ping Windows Defender mặc định block ICMP → luôn thêm `-Pn` khi scan Windows qua proxychains.**

---
## 3. Phát hiện cơ hội Pivot

Sau khi vào được pivot host, kiểm tra network interfaces để tìm các subnet khác:

```bash
ifconfig
# hoặc
ip a
```

Nếu máy có **nhiều NIC**, đó là dấu hiệu cần pivot sang subnet kia.

```
ens192: 10.129.202.64    ← subnet kết nối với attacker
ens224: 172.16.5.129     ← subnet nội bộ, attacker chưa có route
```

---
## Quick Reference

|Kỹ thuật|Lệnh|Dùng khi|
|---|---|---|
|Local Port Forward|`ssh -L <local>:<target_host>:<target_port> user@pivot`|Biết trước dịch vụ cụ thể|
|Dynamic Port Forward|`ssh -D 9050 user@pivot`|Muốn proxy toàn bộ traffic vào subnet|
|Proxychains + Nmap|`proxychains nmap -Pn -sT <target>`|Scan qua SOCKS|
|Proxychains + RDP|`proxychains xfreerdp /v:<ip> /u:<user> /p:<pass>`|RDP vào host nội bộ|

---
## Ghi chú thực tế

- Scan qua proxychains **rất chậm** → chỉ scan từng host hoặc dải IP nhỏ
- Chạy tunnel background để không chiếm terminal: `ssh -D 9050 -f -N ubuntu@<PIVOT_IP>`
