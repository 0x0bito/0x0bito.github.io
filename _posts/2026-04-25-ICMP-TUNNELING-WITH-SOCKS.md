---
title: "ICMP Tunneling with SOCKS"
date: 2026-04-25 18:56:06 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, icmp, socks, tunneling, ptunnel]
description: "HTB CPTS note về ICMP tunneling và SOCKS proxy trong pivoting."
pin: false
---
## Concept

> **Nguyên lý hoạt động ICMP tunneling **đóng gói traffic** bên trong ICMP echo request/response (ping packets).**
> Chỉ hoạt động khi **ping responses được phép** trong firewalled network.

**Attack flow:**

```
Attack Host ──[ICMP]──▶ Jump Box (Pivot) ──[TCP/SSH]──▶ Internal Target
```

**Use cases:**

- Data exfiltration qua firewall
- Tạo pivot tunnel đến external server
- Bypass firewall chặn TCP/UDP nhưng cho phép ICMP

---
## Tool: ptunnel-ng

### Setup trên Attack Host

```bash
# Clone repo
git clone https://github.com/utoni/ptunnel-ng.git

# Build (standard)
sudo ./autogen.sh

# Hoặc build static binary (recommended khi transfer sang target)
sudo apt install automake autoconf -y
cd ptunnel-ng/
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
./autogen.sh
```

> **Lưu ý phiên bản GLIBC Kiểm tra version GLIBC trên target trước khi transfer binary. Mismatch sẽ gây lỗi.**

### Transfer sang Pivot Host

```bash
scp -r ptunnel-ng ubuntu@<PIVOT_IP>:~/
```

---
## Attack Chain

### Bước 1 — Start ptunnel-ng Server (trên Pivot Host)

```bash
# Chạy trên target/pivot host
sudo ./src/ptunnel-ng -r<PIVOT_IP> -R22
```

|Flag|Ý nghĩa|
|---|---|
|`-r`|IP của jump box (reachable từ attack host)|
|`-R`|Port đích sẽ forward đến (22 = SSH)|

**Expected output:**

```
[inf]: Starting ptunnel-ng 1.42.
[inf]: Forwarding incoming ping packets over TCP.
[inf]: Ping proxy is listening in privileged mode.
```

---
### Bước 2 — Connect từ Attack Host (ptunnel-ng Client)

```bash
sudo ./src/ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22
```

|Flag|Ý nghĩa|
|---|---|
|`-p`|IP của ptunnel-ng server (pivot host)|
|`-l`|Local port trên attack host để listen|
|`-r`|Remote IP đích|
|`-R`|Remote port đích|

**Expected output:**

```
[inf]: Relaying packets from incoming TCP streams.
```

---
### Bước 3 — SSH qua ICMP Tunnel

```bash
# SSH thông thường qua tunnel
ssh -p2222 -lubuntu 127.0.0.1
```

Traffic path thực tế:

```
localhost:2222 ──[ICMP]──▶ PIVOT:ptunnel-ng ──[TCP]──▶ PIVOT:22
```

---
### Bước 4 — Dynamic Port Forwarding (Pivot sâu hơn)

```bash
# Tạo SOCKS proxy qua ICMP tunnel
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
```

```bash
# Scan internal network qua proxychains
proxychains nmap -sV -sT 172.16.5.19 -p3389
```

**Full pivot chain:**

```
proxychains ──▶ 127.0.0.1:9050 (SOCKS) ──[ICMP]──▶ Pivot ──▶ Internal Network
```

---
## Verify Traffic (Wireshark)

|Scenario|Traffic captured|
|---|---|
|SSH trực tiếp (`ssh ubuntu@<IP>`)|TCP + SSHv2|
|SSH qua ICMP tunnel (`ssh -p2222 127.0.0.1`)|**ICMP only**|

> **Confirm tunnel hoạt động Check session statistics trên cả client và server side của ptunnel-ng:**
>
> ```
> [inf]: Session statistics:
> [inf]: I/O: 0.00/ 0.00 mb ICMP I/O/R: 248/22/0 Loss: 0.0%
> ```

---
## Quick Reference — Command Summary

```bash
# === PIVOT HOST ===
sudo ./ptunnel-ng -r<PIVOT_IP> -R22

# === ATTACK HOST ===
# 1. Start client
sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22

# 2. SSH qua tunnel
ssh -p2222 -lubuntu 127.0.0.1

# 3. Dynamic port forward
ssh -D 9050 -p2222 -lubuntu 127.0.0.1

# 4. Proxychains scan
proxychains nmap -sV -sT <INTERNAL_IP> -p<PORT>
```

---
