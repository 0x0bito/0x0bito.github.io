---
title: "Attacking Email"
date: 2026-04-19 09:00:00 +0700
categories: [HTB_CPTS, Exploitation]
tags: [email, smtp, imap, pop3, exploitation]
description: "HTB CPTS note về tấn công và enumeration các dịch vụ email."
pin: false
---
## Table of Contents

- 1. Tổng Quan
- 2. Enumeration
    - 2.1 MX Record Lookup
    - 2.2 Port Scanning
- 3. IMAP / POP3 — Protocol Interaction
    - 3.1 Kết Nối với curl và openssl
    - 3.2 Lệnh IMAP Thủ Công
    - 3.3 Lệnh POP3 Thủ Công
- 4. Misconfiguration — Username Enumeration & Dangerous Settings
    - 4.1 SMTP Commands
    - 4.2 POP3 Enumeration
    - 4.3 Tự Động Hóa với smtp-user-enum
    - 4.4 Dangerous Settings — Dovecot
- 5. Cloud Enumeration — Office 365
- 6. Password Attack
    - 6.1 Hydra
    - 6.2 o365spray — Password Spray
- 7. Open Relay Attack

---
## 1. Tổng Quan

**Mail server** nhận/gửi email qua mạng. Luồng cơ bản:

```
[Client] --SMTP--> [Mail Server A] --SMTP--> [Mail Server B] --IMAP/POP3--> [Client]
```

| Giao thức         | Chức năng                                    | Mặc định                        |
| ----------------- | -------------------------------------------- | ------------------------------- |
| SMTP (25/465/587) | Gửi email (client → server, server → server) | Xóa sau khi gửi                 |
| POP3 (143/993)    | Nhận, tải email                              | **Xóa** khỏi server sau khi tải |
| IMAP4 (110/995)   | Nhận, đọc email                              | **Giữ lại** trên server         |


> **Attack surface:** Cloud services (O365, G-Suite) có attack vector khác với self-hosted server. Xác định loại ngay từ đầu bằng MX record.

---
## 2. Enumeration

### 2.1 MX Record Lookup

**Mục đích:** Xác định mail server của target → phân loại cloud hay self-hosted → chọn hướng tấn công.

```bash
# Dùng host
host -t MX <domain>

# Dùng dig
dig mx <domain> | grep "MX" | grep -v ";"

# Resolve IP của mail server
host -t A <mail_server_hostname>
```

**Phân loại kết quả:**

|MX Record chứa|Dịch vụ|Hướng tấn công|
|---|---|---|
|`*.google.com`|G-Suite|CredKing|
|`*.outlook.com` / `*.protection.outlook.com`|Office 365|o365spray, MailSniper|
|`*.zoho.com`|Zoho|Custom tools|
|Domain nội bộ công ty|Self-hosted|SMTP commands, misconfig|

> **Placeholder — Ví dụ thực tế:** `<!-- TODO: thêm output thực tế từ lab -->`

### 2.2 Port Scanning

Dùng khi target **tự host** mail server.

```bash
sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 <target_ip>
```

|Port|Giao thức|Mã hóa|
|---|---|---|
|25|SMTP|Không|
|110|POP3|Không|
|143|IMAP4|Không|
|465|SMTP|TLS|
|587|SMTP|STARTTLS|
|993|IMAP4|TLS|
|995|POP3|TLS|

> **Placeholder — Nmap output mẫu:** `<!-- TODO: paste nmap output từ lab -->`

---
## 3. IMAP / POP3 — Protocol Interaction

**Mục đích:** Sau khi có credentials (từ brute force hoặc leak), dùng IMAP/POP3 để đọc nội dung email — có thể chứa thông tin nhạy cảm, credentials khác, hoặc thông tin nội bộ.

### 3.1 Kết Nối với curl và openssl

```bash
# IMAP — Liệt kê folder
curl -k 'imaps://<FQDN/IP>' --user <user>:<pass>

# IMAP — Đọc email cụ thể trong folder
curl -k 'imaps://<FQDN/IP>/<folder>;UID=<id>' --user <user>:<pass>

# Kết nối thủ công IMAPS (TLS)
openssl s_client -connect <FQDN/IP>:imaps

# Kết nối thủ công POP3S (TLS)
openssl s_client -connect <FQDN/IP>:pop3s
```

### 3.2 Lệnh IMAP Thủ Công

> **Lưu ý:** IMAP yêu cầu mỗi lệnh phải có **tag số** ở đầu (vd: `1`, `2`, `3`...) vì IMAP hỗ trợ gửi nhiều lệnh **song song (async)** → server cần tag để ghép response đúng với lệnh tương ứng.

```
1 LOGIN <user> <pass>
2 LIST "" *           # Liệt kê tất cả folder
3 SELECT INBOX        # Chọn folder (chỉ đọc được nếu EXISTS > 0)
4 FETCH 1 ALL         # Đọc header số 1
5 FETCH 1 BODY[TEXT]  #đọc body
6 LOGOUT
```

### 3.3 Lệnh POP3 Thủ Công

> **Lưu ý:** POP3 **không cần tag** vì chỉ gửi 1 lệnh rồi chờ response tuần tự **(sync)** → không cần phân biệt response.

```
USER <user>
PASS <pass>
LIST          # Liệt kê email
RETR 1        # Đọc email số 1
QUIT
```

> **Placeholder — Kết quả thực tế:** `<!-- TODO: paste session thực tế từ lab -->`

---
## 4. Misconfiguration — Username Enumeration & Dangerous Settings

**Bản chất:** Một số SMTP server cấu hình sai → cho phép truy vấn sự tồn tại của user **mà không cần xác thực**. Dùng để build username list cho bước password spray tiếp theo.

### 4.1 SMTP Commands

Kết nối thủ công:

```bash
telnet <target_ip> 25
```

**VRFY** — Kiểm tra 1 username:

```
VRFY root       → 252 (tồn tại)
VRFY new-user   → 550 (không tồn tại)
```

**EXPN** — Mở rộng mailing list (nguy hiểm hơn VRFY, lộ nhiều user):

```
EXPN support-team → carol@..., elisa@...
```

**RCPT TO** — Giả vờ gửi mail, kiểm tra recipient có hợp lệ không:

```
MAIL FROM:test@htb.com
RCPT TO:john   → 250 OK  (tồn tại)
RCPT TO:julio  → 550     (không tồn tại)
```

> **Lưu ý:** VRFY và EXPN có thể bị disable. RCPT TO thường hoạt động hơn vì khó disable mà không ảnh hưởng chức năng mail.

### 4.2 POP3 Enumeration

```bash
telnet <target_ip> 110

USER john  → +OK  (tồn tại)
USER julio → -ERR (không tồn tại)
```

### 4.3 Tự Động Hóa với smtp-user-enum

```bash
smtp-user-enum -M RCPT -U userlist.txt -D <domain> -t <target_ip>
```

|Flag|Ý nghĩa|
|---|---|
|`-M`|Mode: VRFY / EXPN / RCPT|
|`-U`|File chứa danh sách username|
|`-D`|Domain (để build email: user@domain)|
|`-t`|Target IP|

> **Placeholder — Output mẫu:** `<!-- TODO: paste output thực tế -->`

### 4.4 Dangerous Settings — Dovecot (IMAP/POP3)

Các cấu hình sai phổ biến trên **Dovecot** — IMAP/POP3 server thường gặp:

|Setting|Nguy hiểm vì|
|---|---|
|`auth_debug = yes`|Log toàn bộ thông tin xác thực kể cả password|
|`auth_debug_passwords = yes`|Password bị log dạng **plaintext**|
|`auth_verbose = yes`|Log lần auth thất bại kèm lý do → information disclosure|
|`auth_anonymous_username = anonymous`|Cho phép đăng nhập **ẩn danh**|
|`ssl = no`|Không mã hóa → sniff credentials và nội dung email|
|`disable_plaintext_auth = no`|Cho phép gửi password dạng **plaintext**|

> **Khai thác:** Nếu server có `auth_debug_passwords = yes` và attacker có quyền đọc log → thu thập credentials trực tiếp từ log file.

---
## 5. Cloud Enumeration — Office 365

**Tool:** [o365spray](https://github.com/0xZDH/o365spray)
```

**Workflow:**

```bash
# Bước 1: Xác nhận domain có dùng O365 không
python3 o365spray.py --validate --domain <domain>

# Bước 2: Enumerate username
python3 o365spray.py --enum -U users.txt --domain <domain>
```
---
## 6. Password Attack

### 6.1 Hydra

Dùng cho **self-hosted** SMTP / POP3 / IMAP4.

```bash
# Password spray (1 password, nhiều user)
hydra -L users.txt -p 'Company01!' -f <target_ip> pop3

# Brute force (nhiều password, 1 user)
hydra -l john -P passwords.txt -f <target_ip> smtp
```

|Flag|Ý nghĩa|
|---|---|
|`-L`|File danh sách username|
|`-l`|Username cụ thể|
|`-P`|File danh sách password|
|`-p`|Password cụ thể|
|`-f`|Dừng khi tìm được credential đầu tiên|

### 6.2 o365spray — Password Spray

```bash
python3 o365spray.py --spray \
    -U usersfound.txt \
    -p 'March2022!' \
    --count 1 \
    --lockout 1 \
    --domain <domain>
```

|Flag|Ý nghĩa|
|---|---|
|`--count`|Số password spray mỗi lần trước khi nghỉ|
|`--lockout`|Thời gian chờ giữa các lần (phút) — tránh account lockout|

> ⚠️ **Quan trọng:** Luôn set `--lockout` để tránh lock account. Default lockout thường là 5-10 lần sai.

> **Placeholder — Kết quả thực tế:** `<!-- TODO -->`

---
## 7. Open Relay Attack

**Bản chất:** SMTP server cấu hình sai → cho phép **bất kỳ ai** gửi email qua nó mà không cần xác thực → attacker giả mạo địa chỉ người gửi (spoofing) để phishing.

**Phát hiện Open Relay:**

```bash
nmap -p25 -Pn --script smtp-open-relay <target_ip>
# → Server is an open relay (14/16 tests)
```

**Khai thác với swaks:**

```bash
swaks \
    --from <spoofed_sender@domain.com> \
    --to <target_recipient@domain.com> \
    --header 'Subject: <subject>' \
    --body '<body_with_phishing_link>' \
    --server <open_relay_ip>
```

> **swaks** = Swiss Army Knife for SMTP — tool test SMTP, dùng để gửi email tùy chỉnh qua open relay.

> **Placeholder — Kịch bản phishing thực tế:** `<!-- TODO: mô tả kịch bản nếu gặp trong lab -->`

---
**Attack Flow tổng quát:**

```
MX Record Lookup
    │
    ├─ Cloud (O365/Gmail)
    │       └─ o365spray --validate → --enum → --spray
    │
    └─ Self-hosted
            ├─ Nmap scan ports 25,110,143,465,587,993,995
            │
            ├─ [Username Enum]
            │       ├─ SMTP VRFY/EXPN/RCPT TO
            │       ├─ POP3 USER
            │       └─ smtp-user-enum (tự động hóa)
            │
            ├─ [Password Attack]
            │       └─ Hydra → spray/brute force
            │
            ├─ [Post-Auth — Đọc Email]
            │       ├─ curl imaps:// → liệt kê folder, đọc email
            │       └─ openssl s_client → kết nối thủ công IMAPS/POP3S
            │
            └─ [Exploit Misconfig]
                    └─ smtp-open-relay → swaks → phishing/spoofing
```
