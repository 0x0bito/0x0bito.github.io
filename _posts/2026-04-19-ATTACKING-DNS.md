---
title: "Attacking DNS"
date: 2026-04-19 09:00:00 +0700
categories: [HTB_CPTS, Exploitation]
tags: [dns, exploitation, enumeration, common-services]
description: "HTB CPTS note về enumeration và khai thác dịch vụ DNS."
pin: false
---
> **Port**: 53 (UDP cho query thông thường / TCP cho zone transfer) **Bản chất**: DNS phân giải tên miền thành IP. Trong pentest, DNS là nguồn recon cực kỳ giá trị — lộ toàn bộ infrastructure nếu misconfigured.

---
## Điểm Yếu Cốt Lõi

- **Zone Transfer (AXFR)**: nếu DNS server không giới hạn ai được transfer, attacker download toàn bộ DNS records — lộ tên máy chủ nội bộ, IP, cấu trúc mạng
- **Subdomain Enumeration / Takeover**: brute-force tên subdomain để tìm attack surface ẩn; nếu CNAME trỏ đến dịch vụ hết hạn có thể chiếm subdomain
- **DNS Cache Poisoning (Spoofing)**: inject bản ghi giả vào DNS cache → redirect nạn nhân đến trang giả

---
## Enumeration Ban Đầu

```bash
# Nmap scan DNS service — phát hiện version BIND → tra CVE
nmap -p53 -Pn -sV -sC <TARGET_IP>
```

---
## DNS Recon Cơ Bản

```bash
# Tìm nameservers (NS records)
dig ns <DOMAIN> @<NAMESERVER_IP>

# Tìm mail servers (MX records)
dig mx <DOMAIN> | grep "MX" | grep -v ";"

# Resolve IP của subdomain cụ thể
host -t A <SUBDOMAIN>.<DOMAIN>

# Truy vấn all records
dig ANY <DOMAIN>

# Reverse DNS lookup
dig -x <TARGET_IP>
```

---
## Zone Transfer (AXFR)

> **Bản chất**: AXFR là cơ chế đồng bộ DNS records giữa primary và secondary nameserver. Nếu server không hạn chế, bất kỳ ai cũng có thể request và nhận toàn bộ zone data — lộ nguyên bản đồ mạng nội bộ.

```bash
# Zone Transfer với dig
dig AXFR @<NAMESERVER_IP> <DOMAIN>

# Zone Transfer zone nội bộ
dig AXFR internal.<DOMAIN> @<NAMESERVER_IP>

# Zone Transfer bằng host
host -l <DOMAIN> <NAMESERVER_IP>

# Zone Transfer bằng dnsrecon
dnsrecon -d <DOMAIN> -t axfr

# Fierce — tự dò toàn bộ NS rồi thử AXFR hàng loạt
fierce --domain <DOMAIN>
```

Nếu thành công, output dạng:

```
dev.<DOMAIN>.      A     10.10.10.5
admin.<DOMAIN>.    A     10.10.10.10
vpn.<DOMAIN>.      A     10.10.10.15
```

---
## Subdomain Enumeration

```bash
# Passive + active với subfinder
subfinder -d <DOMAIN> -v

# Brute-force với dnsenum (đệ quy)
dnsenum --enum <DOMAIN> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r

# Brute-force với dnsenum chỉ định nameserver cụ thể
dnsenum --dnsserver <NAMESERVER_IP> --enum -p 0 -s 0 -f <WORDLIST> <DOMAIN>

# Brute-force với gobuster
gobuster dns -d <DOMAIN> -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Subbrute với SecLists wordlist lớn hơn (khuyến nghị cho HTB)
echo "<NAMESERVER_IP>" > ./resolvers.txt
./subbrute/subbrute.py <DOMAIN> -s /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r ./resolvers.txt
```

> **Subbrute** hữu ích khi đang pivot vào internal network — dùng resolver nội bộ thay vì DNS public.

---
## Domain Takeover & Subdomain Takeover

> **Bản chất**: Xảy ra khi một CNAME record trỏ đến dịch vụ bên thứ ba (AWS S3, GitHub Pages, Fastly…) mà dịch vụ đó đã hết hạn hoặc bị xóa. Attacker đăng ký lại dịch vụ đó → kiểm soát toàn bộ subdomain của target.

**Flow kiểm tra**:

```bash
# Bước 1: Enum subdomains → tìm CNAME record
host <SUBDOMAIN>.<DOMAIN>
# Output mẫu: support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com

# Bước 2: Kiểm tra dịch vụ còn tồn tại không
# Truy cập URL → nếu trả về NoSuchBucket / 404 / Not Found → vulnerable

# Bước 3: Đăng ký lại tài nguyên cùng tên trên dịch vụ đó
# Ví dụ: tạo S3 bucket tên "<BUCKET_NAME>" → kiểm soát <SUBDOMAIN>.<DOMAIN>
```

**Reference**: [can-i-take-over-xyz](https://github.com/EdOverflow/can-i-take-over-xyz) — danh sách dịch vụ vulnerable kèm cách kiểm tra.

---
## DNS Spoofing / Cache Poisoning

> **Bản chất**: Inject bản ghi DNS giả vào cache của resolver → nạn nhân query DNS nhận địa chỉ giả → bị redirect đến trang attacker kiểm soát (phishing, credential harvesting).

**Hai con đường chính**:

- **MITM**: chặn giữa victim và DNS server, trả về response giả
- **Exploit DNS server**: khai thác lỗ hổng trên DNS server để sửa trực tiếp record

**Ettercap — Local DNS Cache Poisoning**:

```bash
# Bước 1: Sửa file DNS spoof mapping
nano /etc/ettercap/etter.dns
# Thêm vào:
# <TARGET_DOMAIN>      A   <ATTACKER_IP>
# *.<TARGET_DOMAIN>    A   <ATTACKER_IP>

# Bước 2: Mở Ettercap → Hosts > Scan for Hosts
# Bước 3: Add <VICTIM_IP> vào Target1, <GATEWAY_IP> vào Target2
# Bước 4: Plugins > Manage Plugins > Bật dns_spoof
```

**Bettercap — DNS Spoof**:

```bash
set dns.spoof.domains <TARGET_DOMAIN>
dns.spoof on
```

Sau khi active: mọi query từ `<VICTIM_IP>` đến `<TARGET_DOMAIN>` → trả về `<ATTACKER_IP>` → victim load trang giả.

---
## Dangerous Settings (named.conf)

|Setting|Nguy hiểm vì|
|---|---|
|`allow-transfer { any; };`|Zone Transfer từ mọi IP → lộ toàn bộ DNS records|
|`allow-query { any; };`|Mọi IP query DNS nội bộ|
|`recursion yes;` + public-facing|DNS Amplification / Recursion Abuse|
|`dnssec-validation no;`|Tắt DNSSEC → dễ bị DNS Spoofing|
|`version "BIND 9.x.x";`|Lộ version → tra CVE|

```
allow-transfer { <IP_SECONDARY_DNS>; };  ✅
allow-transfer { any; };                 ☠️
```

---
## Sau Enumeration: Target Thú Vị

Sau zone transfer hoặc subdomain enum, ưu tiên kiểm tra:

|Pattern|Tại sao quan tâm|
|---|---|
|`vpn.*`, `remote.*`|Remote access endpoint|
|`dev.*`, `staging.*`|Môi trường test — thường ít hardened|
|`admin.*`, `manage.*`|Admin panel|
|`mail.*`, `smtp.*`, `mx.*`|Email infrastructure|
|`db.*`, `sql.*`, `mysql.*`|Database có thể exposed|
|`internal.*`, `corp.*`|Có thể có zone nội bộ riêng → thử AXFR thêm|
