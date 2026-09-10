# TryHackMe — TakeOver | Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** [TakeOver](https://tryhackme.com/room/takeover)  
**Çətinlik:** Asan  
**Kateqoriya:** Web, Subdomain Enumeration, Subdomain Takeover  
**Müəllif:** ramin  

---

## 📋 Ümumi Baxış

Bu CTF-də "FutureVera" adlı kosmik araşdırma şirkətinin saytı hədəf götürülür. Şirkət CEO-su bildirir ki, blackhat hackerlər onları **takeover** ilə hədələyir və fidyə istəyir. Məqsəd: hackərlərin hansı subdomaini takeover edə biləcəyini tapmaq.

> *"We believe that the future is in space... Recently blackhat hackers approached us saying they could takeover and are asking us for a big ransom."*

---

## 🔍 Mərhələ 1 — Hədəfi Tanımaq

### /etc/hosts-a əlavə etmək

CTF-in özəl domeini olduğu üçün kompüterimizə bu domenin hansı IP-yə aid olduğunu öyrətməliyik:

```bash
echo "MACHINE_IP futurevera.thm" >> /etc/hosts
```

**Niyə:** Normal internet DNS serveri `futurevera.thm`-i tanımır — çünki bu real domen deyil, yalnız TryHackMe şəbəkəsində mövcuddur.

### Port Skanı

```bash
nmap -sV futurevera.thm
```

**Nəticə:**

| Port | Xidmət | Qeyd |
|------|--------|------|
| 22   | SSH | Uzaqdan giriş |
| 80   | HTTP | Veb sayt |
| 443  | HTTPS | Şifrəli veb sayt |

Saytı brauzerdə açdıqda sadə bir kosmik araşdırma saytı görünür — heç bir flag yoxdur.

---

## 🔍 Mərhələ 2 — Subdomain Axtarışı (FFUF)

### FFUF nədir?

**FFUF (Fuzz Faster U Fool)** — söz siyahısındakı hər sözü avtomatik sınayan alətdir. Bizim halda hər sözü subdomain kimi sınayır.

### Əmr:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -H "Host: FUZZ.futurevera.thm" \
  -u https://futurevera.thm \
  -k \
  -fs 4605
```

| Parametr | Məna |
|----------|------|
| `-w` | Wordlist (söz siyahısı) faylı |
| `FUZZ` | Hər dəfə siyahıdan yeni söz buraya yazılır |
| `-H "Host: FUZZ..."` | Subdomaini server başlığında göndərir |
| `-k` | SSL sertifikat xətalarını ignore et |
| `-fs 4605` | 4605 bayt cavabları filtrələ (yanlış cavablar) |

### Nəticə:

```
support   → 200 OK ✅
blog      → 200 OK ✅
```

İki subdomain tapıldı: `support.futurevera.thm` və `blog.futurevera.thm`

### /etc/hosts-a əlavə etmək:

```bash
echo "MACHINE_IP support.futurevera.thm" >> /etc/hosts
echo "MACHINE_IP blog.futurevera.thm" >> /etc/hosts
```

---

## 🔐 Mərhələ 3 — SSL Sertifikatını Oxumaq

### SSL Sertifikat nədir?

Hər HTTPS saytının bir "şəxsiyyət vəsiqəsi" var — SSL sertifikatı. Bu sertifikatın içində **SAN (Subject Alternative Name)** bölməsi var. Burada sertifikatın qüvvədə olduğu **bütün domainlər** yazılır.

**Vacib məqam:** Şirkətlər bəzən gizli saxlamaq istədikləri subdomainləri SAN bölməsinə yazırlar — amma sertifikat **hər kəsə açıqdır!**

### Əmr:

```bash
openssl s_client -connect support.futurevera.thm:443 \
  </dev/null 2>/dev/null | openssl x509 -noout -text
```

| Hissə | Məna |
|-------|------|
| `openssl s_client -connect` | Serverə SSL ilə qoşul |
| `openssl x509 -noout -text` | Sertifikatı oxu və göstər |

### Nəticə — SAN bölməsində:

```
X509v3 Subject Alternative Name:
    DNS:secrethelpdesk934752.support.futurevera.thm
```

🎯 **Gizli subdomain tapıldı!**

### /etc/hosts-a əlavə etmək:

```bash
echo "MACHINE_IP secrethelpdesk934752.support.futurevera.thm" >> /etc/hosts
```

---

## 💥 Mərhələ 4 — Flagı Tapmaq

### curl ilə müraciət:

```bash
curl -i http://secrethelpdesk934752.support.futurevera.thm
```

**Niyə `http://` — `https://` yox?**  
Çünki SSL sertifikatı bu subdomain üçün uyğun deyil — `https` xəta verir. `http` ilə isə server cavab verir.

**`-i` nə edir?**  
Yalnız səhifə mənbəyini yox, HTTP başlıqlarını da göstərir.

### Nəticə:

```
HTTP/1.1 302 Found
Location: http://flag{beea0d6edfcee06a59b83fb50ae81b2f}.s3-website-us-west-3.amazonaws.com/
```

🚩 **FLAG:** `flag{beea0d6edfcee06a59b83fb50ae81b2f}`

---

## 🧠 Subdomain Takeover — Niyə Təhlükəlidir?

### Nə baş verdi?

```
secrethelpdesk934752.support.futurevera.thm
              ↓
    Amazon S3 bucket-ə yönləndirilmişdi
              ↓
    Şirkət o S3 bucket-i SİLDİ ❌
              ↓
    DNS qeydi hələ DURUR ✅
              ↓
    Haker həmin S3 bucket adını yaradaraq
    subdomaini öz kontroluna keçirə bilər! 😈
```

### Real Təhlükə:

| Hücum | Nəticə |
|-------|--------|
| Phishing | İstifadəçilər şifrələrini hackerin saytına yazır |
| Cookie oğurluğu | Eyni domain olduğu üçün cookie-lərə çatır |
| Reputasiya zərəri | Rəsmi subdomain üzərindən saxta məzmun |

---

## 🗺️ Bütün Prosesin Xəritəsi

```
🎯 Hədəf: futurevera.thm
        ↓
⚙️  /etc/hosts-a əlavə etdik
        ↓
🔍 FFUF ilə subdomainlər axtardıq
        ↓
✅ support.futurevera.thm tapıldı
        ↓
🔐 SSL sertifikatını OpenSSL ilə oxuduq
        ↓
💡 SAN bölməsində gizli subdomain:
   secrethelpdesk934752.support.futurevera.thm
        ↓
🌐 curl -i http:// ilə müraciət etdik
        ↓
🚩 302 Redirect → URL-də FLAG!
   flag{beea0d6edfcee06a59b83fb50ae81b2f}
```

---

## 📚 Öyrənilən Konsepsiyalar

| Konsepsiya | İzah |
|-----------|------|
| **Virtual Hosting** | Bir IP-də çox sayt saxlamaq, Host başlığı ilə fərqləndirilir |
| **FFUF** | Avtomatik subdomain axtarış aləti |
| **SSL/TLS Sertifikat** | Saytın şəxsiyyət vəsiqəsi |
| **SAN (Subject Alternative Name)** | Sertifikatdakı bütün domainlər — gizli subdomainlər burda olur |
| **Subdomain Takeover** | Silinmiş xarici xidmət → DNS qeydi qaldı → Haker ələ keçirir |
| **302 Redirect** | Server "burda deyil, ora get" deyir — ora-da flag var idi |

---

## 🛡️ Müdafiə Üsulları

1. **İstifadəsiz DNS qeydlərini silin** — xidmət silinəndə DNS qeydini də silin
2. **Sertifikat SAN-larını diqqətlə idarə edin** — gizli subdomainləri sertifikata yazmayın
3. **Certificate Transparency loglarını izləyin** — kiminsə sizin adınıza sertifikat aldığını görün
4. **Subdomain monitorinqi** — müntəzəm olaraq bütün subdomainləri yoxlayın

---

## 🏁 Nəticə

Bu CTF-in əsas dərsi: **Gizli saxlamaq istədiyiniz şeyi SSL sertifikatına yazmayın!** Sertifikat hər kəsə açıqdır və SAN bölməsindəki hər subdomain görünür.

**Flag:** `flag{beea0d6edfcee06a59b83fb50ae81b2f}` ✅

---

*Writeup: TryHackMe TakeOver otağı üçün Azərbaycan dilində hazırlanmışdır.*
