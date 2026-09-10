# TryHackMe — Dig Dug Writeup (Azərbaycanca)

---

## 📋 Otaq Haqqında

**Çətinlik:** Easy  
**Mövzu:** DNS enumeration, dig aləti, TXT record  
**Ssenariy:** Hədəf maşın həm normal server həm də DNS serveridir.
`givemetheflag.com` domeni üçün xüsusi DNS sorğusu göndərərək flag tap!

---

## 1. 🌐 DNS Nədir? (Qısa İzah)

DNS = **Domain Name System** — internetin telefon kitabçasıdır.

```
Sən yazırsan:  google.com
DNS deyir:     142.250.74.46
Brauzer gedir: həmin IP-yə
```

**DNS Record Tipləri:**

| Tip | Məna |
|-----|------|
| A | Domain → IPv4 ünvan |
| AAAA | Domain → IPv6 ünvan |
| MX | Mail server |
| **TXT** | İstənilən mətn — **flag burda!** |
| CNAME | Başqa ada yönləndirmə |
| NS | DNS server adı |

---

## 2. 🔍 Kəşfiyyat

**Nmap skanı:**
```bash
nmap -sV -sC 10.81.189.60
```

Maşın həm veb server həm də **DNS server (port 53)** kimi işləyir.

---

## 3. 🎯 DNS Sorğusu — dig

`dig` aləti DNS sorğuları göndərmək üçündür.

**Komanda sintaksisi:**
```bash
dig [domain] [record_tipi] @[DNS_server_IP]
```

**Hər hissənin mənası:**
```
dig              → DNS sorğu aləti
givemetheflag.com → hansı domain haqqında soruşuruq
TXT              → hansı tip məlumat istəyirik
@10.81.189.60    → hansı DNS serverə soruşuruq (@ mütləq lazımdır!)
```

---

## 4. ⚠️ Səhv Komanda

```bash
# SƏHV — @ işarəsi yoxdur!
dig givemetheflag.com TXT 10.81.189.60
```

**Cavab:** `NXDOMAIN` — server tapılmadı!

Səbəb: `@` olmadan `dig` öz default DNS serverinə soruşur,
hədəf maşına deyil!

---

## 5. ✅ Düzgün Komanda

```bash
dig givemetheflag.com TXT @10.81.189.60
```

**Cavab:**
```
;; ANSWER SECTION:
givemetheflag.com.  0  IN  TXT  "flag{0767ccd06e79853318f25aeb08ff83e2}"
```

---

## 6. 🚩 Flag

```
flag{0767ccd06e79853318f25aeb08ff83e2}
```

---

## 7. 🗺️ Attack Chain

```
Hədəf maşın DNS server kimi işləyir
        ↓
givemetheflag.com domeninin TXT record-unda flag var
        ↓
dig givemetheflag.com TXT @10.81.189.60
        ↓
ANSWER SECTION → flag{...} 🚩
```

---

## 8. 📝 Əsas Öyrənilənlər

| Konsept | İzah |
|---------|------|
| **dig** | DNS sorğu aləti |
| **TXT record** | Domenə mətn məlumatı əlavə etmək üçün |
| **@IP** | Xüsusi DNS serverə sorğu göndərmək |
| **NXDOMAIN** | Domain tapılmadı xətası |
| **NOERROR** | Sorğu uğurlu oldu |

---

## 9. 💡 Faydalı dig Komandaları

```bash
# TXT record
dig domain.com TXT @IP

# Bütün recordlar
dig domain.com ANY @IP

# A record (IP ünvan)
dig domain.com A @IP

# Mail server
dig domain.com MX @IP

# DNS server
dig domain.com NS @IP

# Qısa cavab (yalnız nəticə)
dig domain.com TXT @IP +short
```

---

## 10. ⚠️ Mühüm Qeyd

> **`@` işarəsini unutma!**
> `dig domain.com TXT IP` — səhvdir, öz DNS serverinə soruşur
> `dig domain.com TXT @IP` — düzgündür, hədəf serverə soruşur

---

*Hazırladı: CTF Player | TryHackMe — Dig Dug*
