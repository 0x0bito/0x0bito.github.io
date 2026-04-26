---
title: "Information Gathering - Web Edition"
date: 2026-04-11 09:00:00 +0700
categories: [HTB_CPTS, Recon]
tags: [reconnaissance, web, information-gathering, osint]
description: "HTB CPTS note về quy trình information gathering cho mục tiêu web."
pin: false
---
tags:
  - recon
  - web
  - footprinting
---

---

# 🌐 Information Gathering – Web Edition

Web reconnaissance là bước đầu tiên trong mọi pentest engagement — giống như thám tử thu thập manh mối trước khi lập kế hoạch hành động.

---

## 🎯 Mục tiêu chính

|Mục tiêu|Mô tả|
|---|---|
|**Xác định assets**|Tìm tất cả domain, subdomain, IP liên quan|
|**Khám phá thông tin ẩn**|Directory, file, công nghệ không công khai|
|**Phân tích attack surface**|Open ports, services, phiên bản phần mềm|
|**Thu thập intelligence**|Email, nhân viên, công nghệ đang dùng|

---

## 🔍 Passive vs Active Recon

|Loại|Mô tả|Nguy cơ bị phát hiện|Ví dụ|
|---|---|---|---|
|**Active**|Tương tác trực tiếp với target|Cao|Port scan, vuln scan|
|**Passive**|Dùng dữ liệu công khai, không chạm target|Thấp|WHOIS, DNS enum, Google Dork, Wayback Machine|

> **Chiến lược Luôn bắt đầu bằng passive recon để giảm noise trước khi chuyển sang active.**

---

## 📋 WHOIS

**Mục đích:** Tra cứu thông tin đăng ký domain — chủ sở hữu, ngày đăng ký, nameserver, thông tin liên hệ.

```bash
whois example.com
```

> **Lưu ý WHOIS data có thể không chính xác hoặc bị che giấu bởi privacy service. Luôn cross-verify từ nhiều nguồn.**

---

## 🌍 DNS Enumeration

**Mục đích:** Truy vấn DNS records để ánh xạ hạ tầng của target.

### 🛠️ Công cụ DNS Recon

|Tool|Tính năng nổi bật|Dùng khi nào|
|---|---|---|
|`dig`|Query mọi loại record, output chi tiết, tùy chỉnh cao|Manual query, zone transfer, phân tích sâu|
|`nslookup`|Đơn giản, chủ yếu A/AAAA/MX|Quick check cơ bản|
|`host`|Output ngắn gọn|Kiểm tra nhanh A/AAAA/MX|
|`dnsenum`|Tự động hóa, brute-force, zone transfer|Enum subdomain hàng loạt|
|`fierce`|Recursive search, wildcard detection|Thân thiện, tìm subdomain|
|`dnsrecon`|Kết hợp nhiều kỹ thuật, nhiều output format|Enum toàn diện|
|`theHarvester`|OSINT — gom email, info từ nhiều nguồn|Thu thập email/nhân viên liên quan domain|

> **Ưu tiên `dig` là tool quan trọng nhất — linh hoạt và output đầy đủ nhất. Học thành thục trước.**

---

### Các loại DNS record quan trọng

|Record|Mô tả|
|---|---|
|`A`|Hostname → IPv4|
|`AAAA`|Hostname → IPv6|
|`CNAME`|Alias trỏ sang hostname khác|
|`MX`|Mail server của domain|
|`NS`|Authoritative nameserver|
|`TXT`|Dữ liệu text tùy ý (SPF, DKIM, v.v.)|
|`SOA`|Thông tin quản trị DNS zone|
|`SRV`|Hostname + port của một service cụ thể|
|`PTR`|Reverse DNS: IP → Hostname|

---

### Bảng lệnh `dig` đầy đủ

|Lệnh|Mô tả|
|---|---|
|`dig domain.com`|Query A record mặc định|
|`dig domain.com A`|Lấy IPv4|
|`dig domain.com AAAA`|Lấy IPv6|
|`dig domain.com MX`|Tìm mail server|
|`dig domain.com NS`|Tìm authoritative nameserver|
|`dig domain.com TXT`|Lấy TXT record (SPF, DKIM…)|
|`dig domain.com CNAME`|Lấy CNAME record|
|`dig domain.com SOA`|Lấy thông tin quản trị zone|
|`dig domain.com ANY`|Lấy tất cả record _(nhiều server ignore theo RFC 8482)_|
|`dig @1.1.1.1 domain.com`|Dùng DNS server cụ thể (Cloudflare)|
|`dig +trace domain.com`|Xem toàn bộ chuỗi resolution: root → TLD → authoritative|
|`dig -x 192.168.1.1`|Reverse lookup: IP → hostname|
|`dig +short domain.com`|Output ngắn gọn, chỉ lấy kết quả|
|`dig +noall +answer domain.com`|Chỉ hiển thị phần Answer|

> **Dùng trong script `+short` và `+noall +answer` hữu ích khi pipe output sang tool khác hoặc viết script tự động hóa recon.**

---

### Đọc hiểu output của `dig`

```bash
dig google.com
```

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16449
;; flags: qr rd ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             0       IN      A       142.251.47.142

;; Query time: 0 msec
;; SERVER: 172.23.176.1#53(172.23.176.1) (UDP)
;; MSG SIZE  rcvd: 54
```

**Header:**

|Field|Ý nghĩa|
|---|---|
|`status: NOERROR`|Query thành công|
|`qr`|Đây là gói trả lời (Query Response)|
|`rd`|Client yêu cầu recursive lookup|
|`ad`|Resolver xác nhận data xác thực|
|`ANSWER: 1`|Có 1 kết quả trả về|

**Question Section:** Câu hỏi đã gửi đi — _"A record của google.com là gì?"_

**Answer Section:**

|Field|Ý nghĩa|
|---|---|
|`google.com.`|Domain được query|
|`0`|TTL = 0s → không cache kết quả này|
|`IN`|Internet class (chuẩn)|
|`A`|Loại record|
|`142.251.47.142`|**IP address** — đây là thứ cần tìm|

**Footer:** DNS server đã trả lời (`SERVER`), thời gian xử lý (`Query time`), kích thước gói tin (`MSG SIZE`).

> **Rate limit Một số server phát hiện và block nếu query quá nhiều. Chỉ thực hành trên target được phép (HTB lab).**

---

## 🏷️ Subdomain Enumeration

Subdomain có thể chứa: dev server, staging, app cũ chưa được bảo mật, admin panel, v.v.

|Cách tiếp cận|Mô tả|Công cụ|
|---|---|---|
|**Active**|Brute-force, zone transfer|`dnsenum`, `dig axfr`, `gobuster`|
|**Passive**|CT logs, search engine|`crt.sh`, Google Dork|

---

### 🛠️ Công cụ Brute-Force Subdomain

|Tool|Mô tả|
|---|---|
|`dnsenum`|Toàn diện: DNS enum, brute-force, zone transfer, Google scraping, WHOIS|
|`fierce`|Recursive, wildcard detection, giao diện thân thiện|
|`dnsrecon`|Kết hợp nhiều kỹ thuật, nhiều output format|
|`amass`|Tích hợp nhiều nguồn dữ liệu, actively maintained|
|`assetfinder`|Nhẹ, nhanh, ideal cho quick scan|
|`puredns`|Mạnh, lọc kết quả hiệu quả, xử lý được wordlist lớn|

---

### Brute-force subdomain với `dnsenum`

**Khả năng đầy đủ của `dnsenum`:**

|Tính năng|Mô tả|
|---|---|
|DNS Record Enumeration|Lấy A, AAAA, NS, MX, TXT records|
|Zone Transfer|Tự động thử AXFR từ các nameserver tìm được|
|Subdomain Brute-Force|Dò subdomain theo wordlist|
|Google Scraping|Tìm subdomain thêm từ kết quả Google|
|Reverse Lookup|IP → domain, phát hiện website cùng server|
|WHOIS Lookup|Tra cứu thông tin đăng ký domain|

**Lệnh chuẩn với SecLists:**

```bash
dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r
```

|Flag|Ý nghĩa|
|---|---|
|`--enum`|Shortcut bật nhiều option hữu ích cùng lúc|
|`-f <wordlist>`|Đường dẫn wordlist|
|`-r`|Recursive — nếu tìm được subdomain, tiếp tục enum subdomain của subdomain đó|

**Wordlist nên dùng (SecLists):**

```bash
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt  # nhanh, phổ thông
/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt # toàn diện hơn
```

> **Wordlist strategy**
>
> - Không biết gì về target → dùng **general-purpose** wordlist (top 20k)
> - Biết ngành/công nghệ → dùng **targeted** wordlist
> - Có intel từ recon trước → tự tạo **custom** wordlist từ keywords tìm được

---

### Zone Transfer (AXFR) với `dig`

```bash
dig @ns1.example.com example.com axfr
```

> **Zone Transfer Hầu hết server đã khóa AXFR. Nhưng nếu misconfigured → lộ toàn bộ DNS zone (tất cả subdomain + IP). Đây là goldmine nếu thành công!**

### CT Logs với `crt.sh`

```bash
curl -s "https://crt.sh/?q=%25.example.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u
```

> **Giải thích lệnh**
>
> - `%25.` = wildcard `%.` được URL-encode
> - `jq -r '.[].name_value'` → trích xuất tên domain từ JSON
> - `sed 's/\*\.//g'` → bỏ prefix `*.`
> - `sort -u` → sắp xếp và loại trùng lặp

---

## 🏠 Virtual Host Discovery

**Vấn đề:** Một server có một IP nhưng host nhiều website. Scan IP đơn thuần không thấy hết.

**Cơ chế:** Web server (Apache/Nginx/IIS) đọc **HTTP Host header** trong mỗi request để quyết định trả về content của website nào.

```http
GET / HTTP/1.1
Host: www.example1.com     ← server đọc cái này để phân biệt
```

### VHost vs Subdomain

|Tiêu chí|Subdomain|Virtual Host|
|---|---|---|
|**DNS record**|Bắt buộc có|Không bắt buộc|
|**Phát hiện qua DNS enum?**|Có|Không nhất thiết|
|**Ví dụ**|`blog.example.com`|`dev.example.com`, `example2.org`|

> **Điểm mấu chốt VHost không cần DNS record — server vẫn serve được nếu map thủ công trong `/etc/hosts`. Đây là lý do DNS enum không tìm được hết VHost.**

### Ví dụ cấu hình Apache

```apache
# 3 website khác nhau, cùng 1 IP
<VirtualHost *:80>
    ServerName www.example1.com
    DocumentRoot /var/www/example1
</VirtualHost>

<VirtualHost *:80>
    ServerName www.example2.org
    DocumentRoot /var/www/example2
</VirtualHost>

<VirtualHost *:80>
    ServerName www.another-example.net
    DocumentRoot /var/www/another-example
</VirtualHost>
```

### Tại sao quan trọng trong Pentest?

VHost ẩn thường không public, không có DNS record — nhưng vẫn chạy trên server:

```
dev.example.com      → staging server, ít được bảo mật
admin.example.com    → admin panel ẩn
internal.example.com → app nội bộ
```

### VHost Fuzzing – Cách tìm VHost ẩn

```bash
# Gobuster vhost mode
gobuster vhost -u http://10.10.10.5 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain

# ffuf (alternative)
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -u http://10.10.10.5 \
     -H "Host: FUZZ.example.com"
```

|Flag (gobuster)|Ý nghĩa|
|---|---|
|`-u`|Target IP/URL|
|`-w`|Wordlist hostname|
|`--append-domain`|Tự ghép domain vào sau mỗi từ trong wordlist|

Khi tìm được VHost → thêm vào `/etc/hosts`:

```bash
echo "10.10.10.5  dev.example.com" | sudo tee -a /etc/hosts
```

> **Active Recon VHost fuzzing gửi hàng nghìn request → dễ bị phát hiện và log lại. Chỉ thực hành trên target được phép.**

---

## 🕷️ Web Crawling

**Mục đích:** Tự động duyệt toàn bộ cấu trúc website, thu thập link, file, endpoint ẩn.

### Công cụ Web Crawler phổ biến

|Tool|Mô tả|Dùng khi nào|
|---|---|---|
|**Burp Suite Spider**|Crawler tích hợp Burp, map web app, tìm hidden content|Pentest web app toàn diện|
|**OWASP ZAP**|Free, open-source, có spider tự động|Quét lỗ hổng kết hợp crawl|
|**Scrapy**|Python framework, tùy biến cao|Viết spider tùy chỉnh cho recon|
|**Apache Nutch**|Java, scale lớn, crawl hàng triệu trang|Large-scale recon project|

---

### LinkFinder – Tìm endpoint trong JS files

**Mục đích:** Phân tích file JavaScript để tìm **API endpoint, hidden path, REST API** — những thứ ReconSpider hay bỏ sót.

**Cách dùng:**

```bash
# Scan 1 file JS cụ thể
linkfinder -i https://example.com/main.js -o cli

# Scan toàn bộ website (crawl + phân tích JS)
linkfinder -i https://example.com -d -o cli

# Output ra file HTML
linkfinder -i https://example.com/main.js -o results.html
```

|Flag|Ý nghĩa|
|---|---|
|`-i`|Input — URL của JS file hoặc website|
|`-d`|Domain mode — crawl toàn bộ site tìm JS|
|`-o`|Output format: `cli` hoặc `results.html`|

**Output mẫu:**

```
/api/v1/users
/api/v1/admin/settings
/internal/dashboard
/.env
/api/auth/token
```

> **Workflow chuẩn**
>
> 1. Dùng ReconSpider → lấy danh sách `js_files`
> 2. Feed từng JS file vào LinkFinder → tìm API endpoint ẩn
> 3. Test từng endpoint tìm được

> **Kết hợp với curl**
>
> ```bash
> # Lấy list JS từ results.json rồi feed vào LinkFinder
> cat results.json | jq -r '.js_files[]' | while read url; do
>   linkfinder -i "$url" -o cli
> done
> ```

```
http://example.com/robots.txt
```

> **robots.txt File này liệt kê những path mà webmaster **không muốn** index → thường chứa path nhạy cảm!**

---

### ReconSpider – Spider

**Cài đặt Scrapy trên Kali:**

```
python3 ReconSpider.py http://inlanefreight.com
```

**Output — `results.json`:**

|Key|Dữ liệu|Giá trị với Pentest|
|---|---|---|
|`emails`|Email tìm thấy trên domain|Social engineering, phishing|
|`links`|URL nội bộ|Mapping cấu trúc website|
|`external_files`|PDF, DOC, file ngoài|Metadata leak, thông tin nhạy cảm|
|`js_files`|File JavaScript|API endpoint ẩn, hardcoded secret|
|`form_fields`|Form input|Attack surface cho injection|
|`images`|URL ảnh|EXIF metadata, path disclosure|
|`comments`|HTML comment trong source|Credential bị bỏ quên, path nội bộ|

> **JS files & Comments — hay bị bỏ qua**
>
> - **JS files** thường chứa hardcoded API key, endpoint ẩn, logic xác thực
> - **HTML comments** đôi khi chứa credential hoặc path nội bộ mà dev quên xóa

---

## 🔎 Google Dorks (Search Engine Discovery)

Dùng search operator để tìm thông tin nhạy cảm mà không cần chạm vào target — hoàn toàn **passive recon**.

### Bảng operator đầy đủ

|Operator|Mô tả|Ví dụ|
|---|---|---|
|`site:`|Giới hạn theo domain|`site:example.com`|
|`inurl:`|Tìm keyword trong URL|`inurl:login`|
|`filetype:`|Lọc theo loại file|`filetype:pdf`|
|`intitle:`|Tìm trong title trang|`intitle:"confidential report"`|
|`intext:` / `inbody:`|Tìm trong body text|`intext:"password reset"`|
|`cache:`|Xem cached version|`cache:example.com`|
|`link:`|Tìm trang link tới URL|`link:example.com`|
|`related:`|Tìm website tương tự|`related:example.com`|
|`numrange:`|Tìm số trong khoảng|`site:example.com numrange:1000-2000`|
|`allintext:`|Body chứa tất cả từ chỉ định|`allintext:admin password reset`|
|`allinurl:`|URL chứa tất cả từ chỉ định|`allinurl:admin panel`|
|`allintitle:`|Title chứa tất cả từ chỉ định|`allintitle:confidential report 2023`|
|`AND`|Bắt buộc cả hai điều kiện|`site:example.com AND inurl:admin`|
|`OR`|Một trong hai|`"linux" OR "ubuntu"`|
|`NOT`|Loại trừ điều kiện|`site:bank.com NOT inurl:login`|
|`*`|Wildcard — bất kỳ từ nào|`site:x.com filetype:pdf user* manual`|
|`..`|Range số|`site:shop.com "price" 100..500`|
|`" "`|Exact phrase|`"information security policy"`|
|`-`|Loại trừ keyword|`site:news.com -inurl:sports`|

### Google Dorking – Ứng dụng thực tế

**Tìm login page:**

```
site:example.com inurl:login
site:example.com (inurl:login OR inurl:admin)
```

**Tìm file nhạy cảm bị lộ:**

```
site:example.com filetype:pdf
site:example.com (filetype:xls OR filetype:docx)
```

**Tìm file cấu hình:**

```
site:example.com inurl:config.php
site:example.com (ext:conf OR ext:cnf)
```

**Tìm database backup:**

```
site:example.com inurl:backup
site:example.com filetype:sql
```

> **Google Hacking Database (GHDB) Kho lưu trữ hàng nghìn Google Dork mẫu theo category: https://www.exploit-db.com/google-hacking-database**

---

## ⏳ Web Archives – Wayback Machine

**URL:** https://web.archive.org — lưu trữ snapshot website từ năm **1996** đến nay.

### Cách hoạt động

```
Crawling → Archiving → Accessing
```

Bot tự động download toàn bộ trang (HTML, CSS, JS, ảnh) → lưu kèm timestamp → user truy cập bằng cách chọn URL + ngày.

### Tại sao quan trọng trong Pentest?

|Tình huống|Ý nghĩa|
|---|---|
|**Tìm asset ẩn**|Directory, file, subdomain cũ không còn trên site hiện tại|
|**Theo dõi thay đổi**|So sánh snapshot → thấy cấu trúc cũ, tech stack đã thay|
|**Thu thập OSINT**|Nhân viên cũ, chiến lược, công nghệ từng dùng|
|**Tìm lỗ hổng cũ**|Config file, credential bị expose trong phiên bản cũ|
|**Stealth**|Passive hoàn toàn — không chạm vào infrastructure của target|

### Query qua API

```bash
curl -s "https://web.archive.org/cdx/search/cdx?url=example.com&output=json&fl=timestamp,original&collapse=digest" | jq
```

> **Mẹo Website cũ có thể chứa file cấu hình, credentials, hoặc endpoint không còn tồn tại trong phiên bản hiện tại nhưng vẫn accessible.**

---

## 📜 Certificate Transparency (CT) Logs

**Mục đích:** Mọi SSL/TLS certificate được cấp đều phải đăng ký vào **public ledger** — ai cũng có thể xem. Dùng để tìm subdomain mà không cần brute-force.

### CT Logs vs Brute-force

|Brute-force subdomain|CT Logs|
|---|---|
|Đoán tên theo wordlist|Lấy từ record certificate thực tế|
|Bị giới hạn bởi wordlist|Không bị giới hạn|
|Miss subdomain tên khó đoán|Tìm được cả subdomain không còn active|

> **Điểm đặc biệt CT logs lưu cả **certificate cũ/hết hạn** → subdomain đó có thể chạy software lỗi thời → dễ khai thác hơn.**

### 2 Tool tìm kiếm CT Logs

|Tool|Điểm mạnh|Điểm yếu|
|---|---|---|
|**crt.sh**|Free, không cần đăng ký, giao diện web đơn giản|Filter hạn chế|
|**Censys**|Filter mạnh, tìm được IP/cert liên quan, có API|Cần đăng ký|

### Lệnh crt.sh từ terminal

```bash
# Lấy tất cả subdomain
curl -s "https://crt.sh/?q=%25.example.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u

# Lọc chỉ lấy subdomain chứa "dev"
curl -s "https://crt.sh/?q=example.com&output=json" \
  | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' \
  | sort -u
```

---

## 📁 Well-Known URIs

**Đường dẫn chuẩn:** `https://example.com/.well-known/`

Lưu trữ metadata, cấu hình, thông tin bảo mật tại vị trí **cố định, có thể đoán trước** — browser, app, security tool đều biết tìm ở đây.

### Các URI quan trọng

|URI|Mô tả|Giá trị với Pentest|
|---|---|---|
|`security.txt`|Thông tin liên hệ report lỗ hổng|Tìm contact security team|
|`openid-configuration`|Cấu hình OpenID Connect / OAuth 2.0|Chi tiết auth infrastructure|
|`change-password`|URL trang đổi mật khẩu|Xác nhận có auth system|
|`assetlinks.json`|Xác minh ownership digital asset|Tìm app liên kết domain|
|`mta-sts.txt`|Policy bảo mật email SMTP|Hiểu cấu hình mail server|

### `openid-configuration` — Quan trọng nhất

```
https://example.com/.well-known/openid-configuration
```

Trả về JSON chứa toàn bộ cấu hình auth:

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "userinfo_endpoint": "https://example.com/oauth2/userinfo",
  "jwks_uri": "https://example.com/oauth2/jwks",
  "scopes_supported": ["openid", "profile", "email"]
}
```

|Field|Giá trị với Pentest|
|---|---|
|`authorization_endpoint`|Target để test auth bypass|
|`token_endpoint`|Target để test token forgery|
|`userinfo_endpoint`|Test broken access control|
|`jwks_uri`|Phân tích thuật toán mã hóa JWT|
|`scopes_supported`|Hiểu attack surface của auth|

> **Đây là passive recon — chỉ đọc file public, không tương tác với hệ thống. Nhưng thông tin thu được rất chi tiết về auth infrastructure.**

---

## 🤖 Automating Recon – FinalRecon

### Các Framework Recon tự động

|Tool|Mô tả|
|---|---|
|**FinalRecon**|Python, modular, all-in-one: header, WHOIS, SSL, crawl, DNS, subdomain, dir, wayback|
|**Recon-ng**|Framework mạnh, modular, có thể exploit known vulns|
|**theHarvester**|Chuyên gom email, subdomain, host từ search engine, Shodan|
|**SpiderFoot**|OSINT tự động, tích hợp nhiều nguồn dữ liệu|
|**OSINT Framework**|Bộ sưu tập tool và resource OSINT toàn diện|

---

### FinalRecon – Cài đặt

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
./finalrecon.py --help
```

### Bảng flag đầy đủ

|Flag|Mô tả|
|---|---|
|`--url URL`|Target URL|
|`--headers`|Lấy header information|
|`--sslinfo`|Thông tin SSL certificate|
|`--whois`|WHOIS lookup|
|`--crawl`|Crawl target website|
|`--dns`|DNS enumeration (40+ record types)|
|`--sub`|Subdomain enumeration|
|`--dir`|Directory search|
|`--wayback`|Lấy URL từ Wayback Machine|
|`--ps`|Fast port scan|
|`--full`|Full recon — chạy tất cả module|
|`-w W`|Custom wordlist|
|`-e E`|File extensions (vd: txt,xml,php)|
|`-d D`|Custom DNS server (mặc định: 1.1.1.1)|
|`-k K`|Thêm API key (vd: shodan@key)|
|`-o O`|Export format (mặc định: txt)|

### Ví dụ lệnh thực tế

```bash
# Header + WHOIS
./finalrecon.py --headers --whois --url http://inlanefreight.com

# Full recon
./finalrecon.py --full --url http://inlanefreight.com

# Subdomain + DNS
./finalrecon.py --sub --dns --url http://inlanefreight.com

# Directory search với custom wordlist
./finalrecon.py --dir -w /usr/share/seclists/Discovery/Web-Content/common.txt --url http://inlanefreight.com
```

**Output từ `--headers --whois`:**

```
[+] IP Address : 134.209.24.248

[!] Headers :
Server : Apache/2.4.41 (Ubuntu)
Content-Type : text/html; charset=UTF-8
...

[!] Whois Lookup :
Registrar: Amazon Registrar, Inc.
Creation Date: 2019-08-05
Name Server: NS-161.AWSDNS-20.COM
...
```

> **FinalRecon vs Manual FinalRecon đặc biệt mạnh ở **subdomain enum** — tích hợp 8 nguồn dữ liệu: `crt.sh`, `AnubisDB`, `ThreatMiner`, `CertSpotter`, `Facebook API`, `VirusTotal API`, `Shodan API`, `BeVigil API`.**

> **Kali Linux Nếu `pip3 install` bị chặn do PEP 668:**
>
> ```bash
> pip3 install -r requirements.txt --break-system-packages
> ```

---

## 🛠️ Tool Summary

|Tool|Mục đích|Lệnh mẫu|
|---|---|---|
|`whois`|Tra cứu đăng ký domain|`whois example.com`|
|`dig`|Truy vấn DNS / Zone transfer|`dig @ns1.x.com x.com axfr`|
|`dnsenum`|Brute-force subdomain|`dnsenum x.com -f wordlist.txt`|
|`gobuster`|Vhost / directory brute-force|`gobuster vhost -u http://IP -w list.txt`|
|`ffuf`|Vhost fuzzing với filter|`ffuf -w list.txt -u http://IP -H "Host: FUZZ.x.com" -fs 116`|
|`crt.sh`|CT log – passive subdomain enum|`curl -s "https://crt.sh/?q=%25.x.com&output=json"`|
|`LinkFinder`|Tìm API endpoint trong JS files|`python3 linkfinder.py -i https://target.com -d -o cli`|
|`ReconSpider`|Web crawling tự động|`python3 ReconSpider.py http://target.com`|
|`FinalRecon`|All-in-one recon framework|`./finalrecon.py --full --url http://target.com`|
|Wayback Machine|Web archive|https://web.archive.org|

---

## 🔗 Liên kết
- Footprinting
