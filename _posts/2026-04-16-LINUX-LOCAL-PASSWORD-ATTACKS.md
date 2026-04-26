---
title: "Linux Local Password Attacks"
date: 2026-04-16 00:08:00 +0700
categories: [HTB_CPTS, Credentials]
tags: [credentials, linux, lazagne, mimipenguin, firefox, shadow, hashcat, john]
---

> Tổng hợp các kỹ thuật thu thập credentials trên môi trường Linux — từ config files, crontab, log files đến memory và browser credentials.
{: .prompt-info}

> **Credential Sources:** Config files, DB files, bash history, SSH keys, browser credentials, memory.
{: .prompt-info}

---

## Config & Credential Files

```bash
# Tìm config files
for l in $(echo ".conf .config .cnf"); do
  echo -e "\nFile extension: " $l
  find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done

# Grep password trong từng file .cnf
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib"); do
  echo -e "\nFile: " $i
  grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#"
done

# Tìm notes — có .txt và không có extension
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

| Option | Giải thích |
|---|---|
| `2>/dev/null` | Ném **stderr** vào /dev/null — tránh spam terminal |
| `grep -v "lib\|fonts"` | `-v` = **invert match** — loại bỏ path hệ thống không liên quan |
| `grep -v "\#"` | Bỏ qua **dòng comment** trong config file |
| `-type f` | Chỉ tìm **file thường** (bỏ qua directory, symlink) |
| `! -name "*.*"` | Lấy file **không có extension** |

---

## Database & Script Files

```bash
# Tìm database files
for l in $(echo ".sql .db .*db .db*"); do
  echo -e "\nDB File extension: " $l
  find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man"
done

# Tìm script files
for l in $(echo ".py .pyc .pl .go .jar .c .sh"); do
  echo -e "\nFile extension: " $l
  find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share"
done
```

> Scripts thường hardcode credentials để chạy tự động — đặc biệt chú ý `.sh`, `.py`. DB files như `key4.db`, `cert9.db` là credentials store của Firefox.
{: .prompt-tip}

---

## Crontab & SSH Keys

```bash
# Kiểm tra crontab
cat /etc/crontab
ls -la /etc/cron.*/

# Tìm SSH private key theo nội dung
grep -rnw "PRIVATE KEY" /* 2>/dev/null | grep ":1"

# Xem bash history
tail -n5 /home/*/.bash*
```

| Option | Giải thích |
|---|---|
| `grep -r` | **Recursive** — tìm trong tất cả subdirectory |
| `grep -n` | Hiển thị **số dòng** tìm thấy trong file |
| `grep -w` | **Whole word match** |
| `grep ":1"` | Lọc dòng số 1 — header SSH key luôn ở đầu file |
| `tail -n5` | Lấy **5 dòng cuối** — bash history lưu commands gần nhất ở cuối |

| Path | Mô tả |
|---|---|
| `/etc/crontab` | Crontab toàn hệ thống |
| `/etc/cron.d/` | Cron jobs của từng app |
| `/etc/cron.daily/` | Chạy mỗi ngày |
| `/etc/cron.hourly/` | Chạy mỗi giờ |
| `/etc/cron.weekly/` | Chạy mỗi tuần |

> Cronjobs cần credentials để chạy → admin thường nhúng password thẳng vào script → kiểm tra kỹ nội dung các script được gọi bởi cron.
{: .prompt-tip}

---

## Log Files

```bash
for i in $(ls /var/log/* 2>/dev/null); do
  GREP=$(grep "accepted\|session opened\|failure\|failed\|ssh\|password changed\|sudo\|COMMAND\=" $i 2>/dev/null)
  if [[ $GREP ]]; then
    echo -e "\n#### Log file: " $i
    grep "accepted\|session opened\|failure\|failed\|ssh\|password changed\|sudo\|COMMAND\=" $i 2>/dev/null
  fi
done
```

| File | Chứa gì |
|---|---|
| `/var/log/auth.log` | Mọi sự kiện xác thực (Debian) |
| `/var/log/secure` | Tương tự — RedHat/CentOS |
| `/var/log/faillog` | Các lần đăng nhập thất bại |
| `/var/log/syslog` | Hoạt động hệ thống chung |
| `/var/log/mysqld.log` | MySQL — có thể lộ credentials |

---

## Linux Authentication Files

| File | Quyền | Chứa gì |
|---|---|---|
| `/etc/passwd` | World-readable | User info, placeholder `x` |
| `/etc/shadow` | Root only | Password hashes thật |
| `/etc/security/opasswd` | Root only | Password cũ (chống reuse) |

> Nếu `/etc/passwd` bị ghi nhầm write permission → xóa trường password của root → `su` không cần mật khẩu.
{: .prompt-warning}

**Hash format trong `/etc/shadow`:** `$<id>$<salt>$<hash>`

`$1$` = MD5 · `$6$` = SHA-512 · `$y$` = Yescrypt (mặc định Debian hiện đại)

### Unshadow + Crack

```bash
unshadow /etc/passwd /etc/shadow > unshadowed.hashes
hashcat -m 1800 -a 0 unshadowed.hashes rockyou.txt   # SHA-512 ($6$)
hashcat -m 500  -a 0 unshadowed.hashes rockyou.txt   # MD5 ($1$)
```

> Tài khoản có `*` hoặc `!` trong shadow → bị khóa, Hashcat tự bỏ qua.
{: .prompt-tip}

---

## Hunting for Encrypted Files & SSH Keys

```bash
# Tìm file mã hóa theo extension
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*"); do
  echo -e "\nFile extension: " $ext
  find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done

# Tìm SSH private key theo header
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null

# Kiểm tra key có passphrase không
ssh-keygen -yf ~/.ssh/id_rsa
```

| Option | Giải thích |
|---|---|
| `grep -rnE` | `-E` = **regex mở rộng** để match pattern phức tạp |
| `ssh-keygen -yf` | Đọc key — nếu hỏi passphrase → key bị mã hóa → cần crack |

> SSH key không có extension chuẩn → tìm theo `-----BEGIN ... PRIVATE KEY-----` chắc chắn hơn.
{: .prompt-tip}

---

## Memory & Browser Credentials

```bash
# LaZagne
sudo python3 lazagne.py all

# Firefox decrypt
python3.9 firefox_decrypt.py
```

| Tool | Nguồn credentials |
|---|---|
| `mimipenguin` | GNOME keyring, SSH agent — từ memory process đang chạy |
| `lazagne all` | WiFi, SSH, Apache, Docker, KeePass, Firefox, AWS, Git,... |
| `firefox_decrypt` | `logins.json` — plaintext nếu chưa đặt master password |

---

## Checklist thực chiến

```
Files:   config (.cnf/.conf) → database (.db) → notes → scripts → crontab
History: bash_history → log files (/var/log/auth.log, syslog,...)
Memory:  mimipenguin → lazagne all
Browser: logins.json → firefox_decrypt.py hoặc lazagne browsers
Auth:    /etc/shadow → unshadow → hashcat
```
