---
title: "Password Attacks — Wordlists, Hashcat & John the Ripper"
date: 2026-04-16 00:09:00 +0700
categories: [Pentest, Credentials]
tags: [hashcat, john-the-ripper, cracking, wordlist, cewl, rules, mask-attack, bitlocker, pivoting]
---

> **Workflow:** Thu thập hash → Nhận dạng loại hash → Chọn tool & attack mode → Crack → `--show`
{: .prompt-info}

---

## Password Mutations & Custom Wordlists

### CeWL — Website Keyword Wordlist

```bash
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
```

| Option | Giải thích |
|---|---|
| `-d 4` | **Depth** — crawl sâu tối đa 4 cấp link từ URL gốc |
| `-m 6` | **Min word length** — chỉ lấy từ có độ dài ≥ 6 ký tự |
| `--lowercase` | Chuyển toàn bộ từ về chữ thường |
| `-w inlane.wordlist` | **Output file** |

---

### Username Anarchy — Username Wordlist Generation

```bash
# Chuẩn bị file tên
cat names.txt
# Ben Williamson
# Bob Burgerstien
# Jim Stevenson

./username-anarchy -i /home/ltnbob/names.txt > usernames.txt
```

| Option | Giải thích |
|---|---|
| `-i <file>` | **Input file** — mỗi dòng một `Firstname Lastname` |

> **Output ví dụ cho "Jane Doe":** `jane`, `janedoe`, `jane.doe`, `janed`, `j.doe`, `jdoe`, `djane`, `doej`, `doe.jane`,... — tất cả biến thể username phổ biến được sinh tự động.
{: .prompt-info}

> **OSINT sources:** LinkedIn, website công ty, Google dork `"@company.com"` để lấy email → suy ra naming convention trước khi chạy tool.
{: .prompt-tip}

> **Tại sao không chỉ dùng wordlist có sẵn?** Tổ chức thường dùng naming convention riêng (`firstinitiallastname`, `firstname.lastname`,...). Biết convention → giảm tối đa số lần đoán → tránh lockout.
{: .prompt-info}

---

## Hash Identification — hashid

```bash
hashid -j 193069ceb0461e1d40d216e32c79c704
hashid -m '$1$FNr44XZC$wQxY6HHLrgrGX0e1195k.1'
hashid <hash>
```

| Option | Dùng với | Giải thích |
|---|---|---|
| `-j` | John the Ripper | In kèm **JtR `--format`** tương ứng |
| `-m` | Hashcat | In kèm **Hashcat `-m` mode ID** tương ứng |
| _(không flag)_ | Tham khảo | Chỉ liệt kê các loại hash có thể match |

> **Nhận dạng nhanh theo độ dài hex:** `32 ký tự` = MD5 · `40` = SHA1 · `64` = SHA256 · `128` = SHA512. Prefix `$6$` = sha512crypt · `$1$` = md5crypt · `$2*$` = bcrypt.
{: .prompt-tip}

---

## Hash Type Reference

| Hash Type | Hashcat `-m` | JtR `--format` | Nhận dạng |
|---|---|---|---|
| MD5 | `0` | `raw-md5` | 32 ký tự hex |
| SHA1 | `100` | `raw-sha1` | 40 ký tự hex |
| SHA2-256 | `1400` | `raw-sha256` | 64 ký tự hex |
| SHA2-512 | `1700` | `raw-sha512` | 128 ký tự hex |
| MD4 | `900` | `raw-md4` | 32 ký tự hex |
| NTLM | `1000` | `nt` | Dump từ SAM/NTDS/LSASS |
| DCC2 | `2100` | `mscach2` | `$DCC2$10240#user#hash` |
| sha512crypt | `1800` | `sha512crypt` | Linux `$6$` |
| md5crypt | `500` | `md5crypt` | Linux `$1$` |
| bcrypt | `3200` | `bcrypt` | Web app `$2*$` |
| Kerberoast TGS | `13100` | `krb5` | Active Directory |
| BitLocker | `22100` | `bitlocker` | Encrypted disk |
| WPA2 PMKID | `16800` | `wpa2john` | WiFi |
| ZIP | `17200` | `zip` | ZIP archive |
| RAR | `23800` | `rar` | RAR archive |
| PDF | `10500` | `pdf` | PDF document |
| SSH | `22921` | `ssh` | SSH private key |

> **DCC2 vs NTLM:** NTLM ~4,600,000 H/s · DCC2 ~5,500 H/s → **chậm hơn ~800 lần** vì dùng PBKDF2. DCC2 **không dùng được** cho Pass-the-Hash — phải crack ra plaintext mới dùng được.
{: .prompt-warning}

---

## Hashcat — Full Reference

### Cú pháp chung

```bash
hashcat -a <attack_mode> -m <hash_type> <hashes> [wordlist/rule/mask]
```

| Tham số | Giải thích |
|---|---|
| `-a` | **Attack mode** |
| `-m` | **Hash type ID** |
| `<hashes>` | Hash string đơn hoặc file chứa nhiều hashes |
| `-o <file>` | **Output file** — lưu kết quả dạng `hash:plaintext` |
| `--show` | Hiển thị kết quả đã crack từ **potfile** |
| `--force` | Bỏ qua cảnh báo (dùng trên VM thiếu GPU) |

---

### Attack Mode 0 — Dictionary Attack

```bash
hashcat -a 0 -m 0 e3e3ec5831ad5e7288241960e5d4fdb8 /usr/share/wordlists/rockyou.txt

# Với rule
hashcat -a 0 -m 0 <hash> /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Generate mutated wordlist (không crack, chỉ xuất wordlist)
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

| Option | Giải thích |
|---|---|
| `-a 0` | **Dictionary attack** |
| `-r <rulefile>` | Áp dụng **rule** biến thể từ |
| `--stdout` | Xuất wordlist đã mutate ra terminal |

**Rule files tại `/usr/share/hashcat/rules/`:**

| Rule file | Mô tả |
|---|---|
| `best64.rule` | 64 biến thể phổ biến nhất — dùng trước tiên |
| `rockyou-30000.rule` | 30,000 rules mạnh, coverage rộng |
| `dive.rule` | Rule lớn nhất, toàn diện nhất |
| `d3ad0ne.rule` | Rule phổ biến trong cộng đồng |
| `leetspeak.rule` | Biến thể leet: `a→@`, `e→3`, `o→0`,... |
| `toggles5.rule` | Toggle hoa/thường các ký tự |

> **Quy trình thực tế:** Thử wordlist thuần → nếu fail → thêm `best64.rule` → nếu fail → thử `rockyou-30000.rule` hoặc `dive.rule`.
{: .prompt-tip}

#### Viết Custom Rule

| Function | Tác dụng | Ví dụ với `password` |
|---|---|---|
| `:` | Giữ nguyên | `password` |
| `c` | Viết hoa chữ đầu | `Password` |
| `u` | Uppercase toàn bộ | `PASSWORD` |
| `l` | Lowercase toàn bộ | `password` |
| `so0` | Thay `o` → `0` | `passw0rd` |
| `sa@` | Thay `a` → `@` | `p@ssword` |
| `$!` | Thêm `!` vào cuối | `password!` |

> **Cú pháp kết hợp:** Nhiều function trên **cùng một dòng** = áp dụng đồng thời. Ví dụ: `c so0 sa@ $!` → `P@ssw0rd!`
{: .prompt-info}

```bash
cat custom.rule
:             # password
c             # Password
so0           # passw0rd
c so0         # Passw0rd
sa@           # p@ssword
c sa@         # P@ssword
c sa@ so0     # P@ssw0rd
$!            # password!
$! c          # Password!
$! so0        # passw0rd!
$! sa@        # p@ssword!
$! c so0      # Passw0rd!
$! c sa@      # P@ssword!
$! so0 sa@    # p@ssw0rd!
$! c so0 sa@  # P@ssw0rd!
```

> **OSINT → Custom Wordlist:** Thu thập thông tin nạn nhân → tạo wordlist gốc → áp rule → xác suất trúng cao hơn rockyou.txt rất nhiều.
{: .prompt-tip}

---

### Attack Mode 3 — Mask Attack

```bash
hashcat -a 3 -m 0 1e293d6912d074c0fd15844d803400dd '?u?l?l?l?l?d?s'
hashcat -a 3 -m 0 <hash> -1 ?l?d '?1?1?1?1?1?1?1?1'
```

| Symbol | Charset |
|---|---|
| `?l` | `abcdefghijklmnopqrstuvwxyz` |
| `?u` | `ABCDEFGHIJKLMNOPQRSTUVWXYZ` |
| `?d` | `0123456789` |
| `?h` | `0123456789abcdef` |
| `?H` | `0123456789ABCDEF` |
| `?s` | `!"#$%&'()*+,-./:;<=>?@[]^_` |
| `?a` | `?l?u?d?s` (tất cả printable) |
| `?b` | `0x00 - 0xff` (toàn bộ byte) |

> **Khi nào dùng Mask Attack?** Khi biết **pattern** của mật khẩu: độ dài, loại ký tự, format cố định (VD: `Company2024!`). Custom charset: dùng `-1`, `-2`, `-3`, `-4` để định nghĩa bộ ký tự riêng, tham chiếu bằng `?1`, `?2`, `?3`, `?4`.
{: .prompt-tip}

---

## John the Ripper — Full Reference

### Cracking Modes

```bash
john --single passwd
john --wordlist=rockyou.txt hash.txt
john --wordlist=rockyou.txt --rules hash.txt
john --incremental hash.txt
john --format=raw-md5 hash.txt
john --format=nt hash.txt
john hash.txt --show
```

| Mode | Giải thích | Dùng khi |
|---|---|---|
| `--single` | Sinh candidate từ username, homedir, GECOS | Target Linux, có file passwd |
| `--wordlist` | Dictionary attack | Có wordlist sẵn |
| `--rules` | Biến thể wordlist | Kết hợp với `--wordlist` |
| `--incremental` | Brute-force thông minh (Markov chains) | Không có wordlist |
| `--format` | Chỉ định loại hash | JtR không tự nhận diện đúng |
| `--show` | Hiển thị passwords đã crack từ **john.pot** | Sau khi crack xong |

> **Incremental Mode** rất tốn tài nguyên — nên tùy chỉnh bộ ký tự và độ dài trong `john.conf` để tối ưu.
{: .prompt-warning}

---

### *2john Converters

```bash
python3 ssh2john.py SSH.private > ssh.hash
office2john.py Protected.docx > protected-docx.hash
pdf2john.pl PDF.pdf > pdf.hash
zip2john ZIP.zip > zip.hash
rar2john archive.rar > rar.hash
bitlocker2john -i Backup.vhd > backup.hashes
keepass2john Database.kdbx > keepass.hash
gpg2john private.key > gpg.hash
wpa2john capture.pcap > wpa.hash

john --wordlist=rockyou.txt ssh.hash
john ssh.hash --show
```

| Tool | Dùng cho |
|---|---|
| `ssh2john` | SSH private key |
| `office2john` | MS Office có mật khẩu |
| `pdf2john` | PDF có mật khẩu |
| `zip2john` | ZIP có mật khẩu |
| `rar2john` | RAR có mật khẩu |
| `keepass2john` | KeePass database |
| `bitlocker2john` | Ổ đĩa BitLocker |
| `gpg2john` | File GPG được mã hóa |
| `wpa2john` | WiFi WPA/WPA2 handshake |

> **Quy trình chuẩn với John:** `*2john` → tạo `.hash` → `john --wordlist` → `john --show`
{: .prompt-tip}

---

### Hash Format phổ biến với `--format`

| Format | Mô tả |
|---|---|
| `raw-md5` | Raw MD5 |
| `raw-sha1` | Raw SHA1 |
| `raw-sha256` | Raw SHA256 |
| `raw-sha512` | Raw SHA512 |
| `nt` | Windows NTLM |
| `sha512crypt` | Linux `$6$` |
| `md5crypt` | Linux `$1$` |
| `zip` | ZIP archive |
| `rar` | RAR archive |
| `pdf` | PDF document |
| `ssh` | SSH private key |
| `krb5` | Kerberos 5 |

---

## AES Brute Force

```bash
file GZIP.gzip

for i in $(cat rockyou.txt); do
  openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz
done
```

| Option | Giải thích |
|---|---|
| `enc -aes-256-cbc` | Dùng thuật toán **AES-256 CBC mode** |
| `-d` | **Decrypt mode** |
| `-k $i` | **Password/Key** — thử từng từ trong rockyou.txt |
| `tar xz` | Giải nén output nếu decrypt thành công |

> Sẽ thấy nhiều dòng `gzip: stdin: not in gzip format` — bỏ qua, đây là lỗi khi sai password.
{: .prompt-info}

---

## Mount BitLocker trên Linux — dislocker

```bash
sudo apt-get install dislocker
sudo mkdir -p /media/bitlocker
sudo mkdir -p /media/bitlockermount

# Crack hash
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash
hashcat -a 0 -m 22100 backup.hash /usr/share/wordlists/rockyou.txt

# Kiểm tra partition
sudo losetup -f -P Backup.vhd
lsblk | grep loop

# Mount
sudo dislocker /dev/loop0p1 -u<password> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount

cd /media/bitlockermount && ls -la

# Unmount
sudo umount /media/bitlockermount
sudo umount /media/bitlocker
```

| Lệnh | Giải thích |
|---|---|
| `bitlocker2john -i` | Extract **4 loại hash** từ VHD |
| `grep "bitlocker\$0"` | Lọc chỉ lấy **password hash** |
| `-m 22100` | Hashcat mode cho **BitLocker** |
| `losetup -f -P` | Gắn VHD thành **loop device** |
| `-u<password>` | Password đã crack |
| `mount -o loop` | Mount `dislocker-file` như ổ đĩa thường |

> **4 loại hash của BitLocker:** `$bitlocker$0$` và `$bitlocker$1$` → Password hash → crack được. `$bitlocker$2$` và `$bitlocker$3$` → Recovery key → không thực tế để crack.
{: .prompt-info}

> BitLocker crack rất chậm — ~25 H/s ngay cả trên CPU tốt. Cần GPU hoặc thời gian dài.
{: .prompt-warning}

---

## Pivoting

> **Luồng tấn công:** `Attacker → [SSH SOCKS Proxy :9050] → DMZ Host → Internal Network`
{: .prompt-info}

```bash
ssh -D 9050 user@<DMZ01>
sudo vim /etc/proxychains.conf        # thêm: socks4 127.0.0.1 9050
sudo proxychains -q nmap -sT -Pn 172.16.119.13 --open
proxychains xfreerdp /v:<ip> /u:htb-student /p:HTB_@cademy_stdnt!
```

| Option | Giải thích |
|---|---|
| `ssh -D 9050` | **Dynamic port forwarding** — tạo SOCKS proxy trên `localhost:9050` |
| `proxychains -q` | `-q` = **Quiet** — tắt verbose output |
| `nmap -sT` | **TCP Connect scan** — bắt buộc dùng với proxy |
| `nmap -Pn` | **Skip ping** — firewall thường block ICMP |
| `nmap --open` | Chỉ hiển thị **port đang mở** |

**ProxyChains config `/etc/proxychains.conf`:**

```
[ProxyList]
socks4  127.0.0.1  9050
```
