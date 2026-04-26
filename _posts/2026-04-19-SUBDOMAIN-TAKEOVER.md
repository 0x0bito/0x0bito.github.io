---
title: "Subdomain Takeover"
date: 2026-04-19 09:00:00 +0700
categories: [HTB_CPTS, Exploitation]
tags: [subdomain-takeover, dns, web-security, reconnaissance]
description: "HTB CPTS note về subdomain takeover và CNAME dangling records."
pin: false
---
> **Bản chất**: Công ty hủy dịch vụ bên thứ ba (AWS S3, GitHub Pages, Heroku...) nhưng **quên xóa CNAME record** trong DNS. Attacker đăng ký lại dịch vụ đó → kiểm soát subdomain của công ty mà không cần hack DNS.

---
## Table of Contents

1. [Tại Sao Nguy Hiểm](#tại-sao-nguy-hiểm)
2. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
3. [Flow Tấn Công](#flow-tấn-công)
4. [Fingerprint Dấu Hiệu Vulnerable](#fingerprint-dấu-hiệu-vulnerable)
5. [Tool Tự Động](#tool-tự-động)
6. [References](#references)

---
## Tại Sao Nguy Hiểm

- **Phishing hiệu quả**: victim thấy `customer-drive.inlanefreight.com` → tin tưởng vì subdomain của công ty thật
- **Cookie stealing**: subdomain cùng domain gốc → có thể đọc cookie shared
- **CSRF / CORS bypass**: origin được coi là trusted
- **CSP bypass**: content security policy thường whitelist subdomain cùng domain

> RedHuntLabs (2020): 424,120 / 220 triệu domain vulnerable — 62% thuộc e-commerce, 139 trong Alexa Top 1000.

---
## Cơ Chế Hoạt Động

**Giai đoạn 1 — Chiếm subdomain:**

| Bước | Điều xảy ra |
|---|---|
| 1 | Tìm subdomain có CNAME trỏ đến dịch vụ đã bị xóa |
| 2 | Đăng ký lại tên đó trên dịch vụ bên thứ ba (tạo S3 bucket cùng tên...) |
| 3 | CNAME record cũ của công ty tự động trỏ về server attacker |
| 4 | Attacker kiểm soát hoàn toàn nội dung subdomain |

**Giai đoạn 2 — Exploit victim:**

| Bước | Điều xảy ra |
|---|---|
| 5 | Victim gõ URL subdomain vào browser |
| 6 | DNS server tra cứu → thấy CNAME record cũ → redirect về server attacker |
| 7 | DNS coi subdomain là **legitimate** vì record do chính công ty tạo ra |
| 8 | Victim bị forward đến server attacker mà không hay biết |

---
## Flow Tấn Công

**Bước 1 — Enum subdomains:**
```bash
# Passive enum
subfinder -d <DOMAIN> -v

# Brute-force với subbrute (internal network)
echo "<NAMESERVER_IP>" > resolvers.txt
./subbrute/subbrute.py <DOMAIN> \
    -s /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
    -r ./resolvers.txt
```

**Bước 2 — Tìm CNAME record:**
```bash
# Check CNAME từng subdomain
for sub in $(cat subdomains.txt); do
    CNAME=$(dig CNAME $sub +noall +answer 2>/dev/null)
    if [[ -n "$CNAME" ]]; then
        echo "$CNAME"
    fi
done

# Check thủ công
host <SUBDOMAIN>.<DOMAIN>
# Output mẫu: support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com
```

**Bước 3 — Xác nhận vulnerable:**
```bash
# Truy cập URL trong browser hoặc dùng curl
curl -s https://<SUBDOMAIN>.<DOMAIN>
# Nếu thấy fingerprint lỗi → vulnerable (xem bảng bên dưới)
```

**Bước 4 — Chiếm subdomain:**
```bash
# Ví dụ với AWS S3
# Tạo bucket tên trùng với subdomain
aws s3 mb s3://<BUCKET_NAME>
# → support.inlanefreight.com giờ trỏ về bucket của attacker
```

---
## Fingerprint Dấu Hiệu Vulnerable

| Dịch vụ | CNAME pattern | Lỗi trả về khi bị xóa |
|---|---|---|
| AWS S3 | `*.s3.amazonaws.com` | `NoSuchBucket` |
| GitHub Pages | `*.github.io` | `404 - There isn't a GitHub Pages site here` |
| Azure | `*.azurewebsites.net` | `404 Web Site not found` |
| Heroku | `*.herokuapp.com` | `No such app` |
| Fastly | `*.fastly.net` | `Fastly error: unknown domain` |
| Shopify | `*.myshopify.com` | `Sorry, this shop is currently unavailable` |
| Tumblr | `domains.tumblr.com` | `There's nothing here` |
| Pantheon | `*.pantheonsite.io` | `404 error unknown site` |

---
## Tool Tự Động

```bash
# subjack — quét hàng loạt, phát hiện vulnerable CNAME
go install github.com/haccer/subjack@latest
subjack -w subdomains.txt -t 100 -o vulnerable.txt -ssl

# subzy — nhẹ hơn, cập nhật fingerprint thường xuyên hơn
go install github.com/PentestPad/subzy@latest
subzy run --targets subdomains.txt
```

---
## References

- [can-i-take-over-xyz](https://github.com/EdOverflow/can-i-take-over-xyz) — danh sách đầy đủ dịch vụ vulnerable + fingerprint + hướng dẫn kiểm tra
- [HackerOne Subdomain Takeover reports](https://hackerone.com/hacktivity?querystring=subdomain+takeover) — case thực tế đã được trả bounty
