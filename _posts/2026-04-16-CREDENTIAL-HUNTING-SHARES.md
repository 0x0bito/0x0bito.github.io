---
title: "Credential Hunting in Network Shares"
date: 2026-04-16 00:07:00 +0700
categories: [Pentest, Credentials]
tags: [credentials, smb, snaffler, manspider, netexec, active-directory, shares]
---

> Network shares trong môi trường doanh nghiệp thường chứa credentials bị để lộ vô tình trong file config, script, backup.
{: .prompt-info}

---

## Bước 1 — Enumerate Shares & Permissions

> Biết share nào **có quyền đọc** trước khi quét → tiết kiệm thời gian, không quét vào share bị denied. MANSPIDER sẽ login thành công nhưng kết thúc ngay nếu không có quyền đọc.
{: .prompt-tip}

**Từ Linux:**

```bash
nxc smb <IP> -u <user> -p <pass> --shares
```

**Từ Windows:**

```powershell
net view \\DC01.inlanefreight.local
Get-ADUser -Filter * | Select SamAccountName, Enabled | Format-Table -AutoSize
net user /domain
```

---

## Pattern cần tìm

| Loại | Ví dụ |
|---|---|
| **Keyword trong file** | `passw`, `user`, `token`, `key`, `secret` |
| **Extension đáng ngờ** | `.ini`, `.cfg`, `.env`, `.ps1`, `.bat`, `.xlsx` |
| **Tên file đáng ngờ** | `config`, `cred`, `backup`, `initial`, `passw` |
| **Domain string** | `INLANEFREIGHT\` |
| **Ngôn ngữ địa phương** | Công ty Đức → `Benutzer` thay vì `User` |

> Ưu tiên share của **IT employees** — khả năng chứa credentials cao hơn nhiều. Quét **tất cả shares** có thể truy cập, không bỏ sót NETLOGON/SYSVOL.
{: .prompt-tip}

---

## 🪟 Hunting từ Windows

### Snaffler

> Tool C#, chạy trên máy **domain-joined**, tự động tìm shares có thể truy cập và quét file theo các rule định sẵn.
{: .prompt-info}

```cmd
Snaffler.exe -s
Snaffler.exe -s -u
Snaffler.exe -s -i IT
Snaffler.exe -s -o results.log
```

| Flag | Ý nghĩa |
|---|---|
| `-s` | Chạy scan cơ bản |
| `-u` | Lấy user list từ AD, tìm reference trong file |
| `-i <share>` | Chỉ quét share được chỉ định |
| `-n <share>` | Loại trừ share khỏi scan |
| `-o <file>` | Lưu output ra file |

**Màu sắc output:**

| Màu | Ý nghĩa |
|---|---|
| 🔴 Red | Rất có khả năng chứa credentials → xem ngay |
| 🟡 Yellow | Đáng ngờ, cần kiểm tra thủ công |
| 🟢 Green | Share có thể đọc được |
| ⚫ Black | Không có quyền truy cập |

> Output rất nhiều — phần lớn là **false positives** từ tool scripts dùng `-PassThru`, `-Password` làm parameter. Chỉ focus vào **Red findings**.
{: .prompt-warning}

> `unattend.xml` trong `\Windows\Panther\` thường chứa `AdministratorPassword` từ quá trình cài đặt Windows tự động.
{: .prompt-tip}

---

### PowerHuntShares

> PowerShell script, **không cần** domain-joined. Tạo **HTML report** trực quan sau khi quét xong.
{: .prompt-info}

```powershell
Import-Module .\PowerHuntShares.psm1
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public
Start-Process "C:\Users\Public\SmbShareHunt-*.html"
Import-Csv ".\Results\*-Shares-Interesting-Files.csv" | Select ShareName, FileName, FilePath | Format-Table -AutoSize
```

> Chỉ quét **tối đa 3 cấp thư mục** → bỏ sót file nằm sâu hơn. Dùng MANSPIDER để quét không giới hạn depth.
{: .prompt-warning}

---

## 🐧 Hunting từ Linux

### MANSPIDER

> Quét SMB shares từ Linux, không cần trong domain, không giới hạn depth. Là tool tốt nhất để quét toàn bộ target trong một lệnh.
{: .prompt-info}

```bash
# Quét theo content
manspider <IP> -u '<user>' -p '<pass>' -t 20 -c 'passw' 'auth' 'backup'

# Quét theo tên file
manspider <IP> -f 'passw' 'cred' 'config' 'backup' 'secret' 'auth' 'tunnel' 'vpn' \
  -u '<user>' -p '<pass>' -t 20

# Quét share cụ thể
manspider <IP> -u '<user>' -p '<pass>' -t 20 -c 'passw' 'auth' 'backup' --share <ShareName>

# Quét theo extension
manspider <IP> -e ps1 bat cfg env txt -u '<user>' -p '<pass>' -t 20
```

| Flag | Ý nghĩa |
|---|---|
| `-c 'passw'` | Tìm **content** chứa chuỗi |
| `-f 'cred'` | Tìm theo **tên file** |
| `-e ps1 bat` | Tìm theo **extension** |
| `-t 20` | Số threads |

**Kết quả lưu tại:** `./manspider/loot/`

---

### NetExec Spider

```bash
# Quét content trong share cụ thể
nxc smb <ip> -u <username> -p <passwd> \
  --spider $share \
  --content \
  --pattern "passw|cred|config|backup|secret|auth|admin"

# Quét nhiều shares bằng loop
for share in HR Company IT NETLOGON SYSVOL; do
  echo "=== $share ==="
  nxc smb <IP> -u <user> -p <pass> \
    --spider $share \
    --content \
    --pattern "passw|cred|config|backup|secret|auth|admin"
done
```

| Flag | Ý nghĩa |
|---|---|
| `--spider <share>` | Tên share cần quét |
| `--content` | Đọc nội dung file |
| `--pattern <str>` | Chuỗi cần tìm |
