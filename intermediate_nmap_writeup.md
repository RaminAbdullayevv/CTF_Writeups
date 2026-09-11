# TryHackMe — Intermediate Nmap Writeup (Azərbaycan dilində)

**Otaq linki:** https://tryhackme.com/room/intermediatenmap  
**Çətinlik:** Asan  
**Mövzu:** Nmap + Netcat + SSH

---

## Ümumi Baxış

Bu lab-da öyrəndiyimiz nmap bacarıqlarını **netcat** və **SSH** ilə birləşdiririk. Hədəf maşın yüksək bir portda məlumat yayımlayır — həmin məlumatdan istifadə edərək SSH ilə login olub flag-i tapırıq.

---

## Alətlər

- **nmap** — port scan üçün
- **netcat (nc)** — yüksək porta qoşulmaq üçün
- **ssh** — uzaqdan giriş üçün

---

## Addım 1: Nmap ilə Scan

AttackBox-da terminal aç və bu komandanı işlət:

```bash
nmap -sC -sV -p- -T4 <MACHINE_IP>
```

**Parametrlərin izahı:**
- `-sC` — default skriptləri işlət
- `-sV` — servis/versiya aşkarla
- `-p-` — bütün 65535 portu scan et
- `-T4` — sürətli scan

**Nəticə — 3 açıq port tapılır:**

| Port | Servis | Qeyd |
|------|--------|------|
| 22/tcp | SSH | Normal SSH portu |
| 2222/tcp | SSH | Alternativ SSH portu |
| 31337/tcp | Elite? | Yüksək port — şübhəli! |

---

## Addım 2: Yüksək Porta Qoşul (Port 31337)

Port 31337-də nə olduğunu görmək üçün **netcat** ilə qoşul:

```bash
nc <MACHINE_IP> 31337
```

**Nəticə:** Bu port sənə birbaşa **username və password** verir:

```
user: ubuntu
pass: Dafdas!!/str0ng
```

> **Qeyd:** Port 31337 hacker dünyasında "Elite" portu kimi tanınır — bu məşhur bir easter egg-dir!

---

## Addım 3: SSH ilə Login Ol

Əldə etdiyimiz credentials ilə SSH-a qoşuluruq:

```bash
ssh ubuntu@<MACHINE_IP>
```

Şifrəni soruşanda: `Dafdas!!/str0ng` daxil et

> **Qeyd:** Port 2222 işləmir, yalnız port **22** qəbul edir.

---

## Addım 4: Flag-i Tap

Login olduqdan sonra:

```bash
ls -al
```

Heç nə yoxdursa digər istifadəçilərə bax:

```bash
ls /home/
cd /home/<digər_user>/
cat flag.txt
```

**Flag:**
```
flag{251f309497a18888dde5222761ea88e4}
```

---

## Öyrəndiklərimiz

| Alət | İstifadəsi |
|------|-----------|
| `nmap -sC -sV -p-` | Bütün portları servis məlumatı ilə scan et |
| `nc IP PORT` | Porta qoşulub məlumat al |
| `ssh user@IP` | SSH ilə uzaqdan giriş |
| `ls /home/` | Sistemdəki digər istifadəçiləri gör |

---

## Əsas Dərs

> **Enumeration (siyahıya alma) hər şeydir!**  
> Nə qədər ətraflı scan etsən, bir o qədər çox məlumat tapırsan.  
> Bu halda nmap-in `--script` nəticələri birbaşa username/password-u üzə çıxardı.

---

*Writeup Azərbaycan dilində hazırlanmışdır.*
