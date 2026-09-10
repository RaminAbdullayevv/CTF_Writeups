# TryHackMe — CyberHeroes Writeup (Azərbaycanca)

---

## 📋 Otaq Haqqında

**Çətinlik:** Easy  
**Mövzu:** Client-side authentication, JavaScript analizi, Source code review  
**Ssenariy:** CyberHeros adlı bir hacker qrupu var. Login səhifəsindəki
vulnerability-ni tap və daxil ol!

---

## 1. 🔍 Kəşfiyyat (Nmap)

```bash
nmap -sV -sC 10.82.167.31
```

**Tapılan portlar:**

| Port | Servis | Versiya |
|------|--------|---------|
| 22   | SSH    | OpenSSH |
| 80   | HTTP   | Apache 2.4.48 (Ubuntu) |

---

## 2. 🌐 Veb Saytı Kəşf Et

```
http://10.82.167.31
```

Saytda yazılıb:
```
"find the vuln on our login page and login to join us"
```

Login səhifəsinə get:
```
http://10.82.167.31/login.html
```

---

## 3. 📁 Directory Listing

```
http://10.82.167.31/assets/
```

**Directory listing açıqdır!** Qovluqlar görünür:
- `/css/`
- `/img/`
- `/js/`
- `/vendor/`

---

## 4. 🔎 Source Code Analizi

Login səhifəsinin source koduna baxdıq:

```
Ctrl+U  (brauzerdə source kod)
```

**Tapılan JavaScript kodu:**

```javascript
function authenticate() {
  a = document.getElementById('uname')
  b = document.getElementById('pass')
  const RevereString = str => [...str].reverse().join('');
  if (a.value=="h3ck3rBoi" & b.value==RevereString("54321@terceSrepuS")) { 
    // flag göstər
  }
}
```

> ⚠️ **Böyük Səhv!** Authentication **client-side** (brauzerdə) edilir!
> Yəni şifrə yoxlaması serverdə deyil, **brauzerin özündə** baş verir.
> Bu o deməkdir ki, hər kəs source koda baxıb şifrəni tapa bilər!

---

## 5. 🔓 Şifrəni Decode Et

Source kodda şifrə **tərsinə yazılmışdı:**

```javascript
RevereString("54321@terceSrepuS")
```

`RevereString` funksiyası stringi tərsinə çevirir:

```
54321@terceSrepuS → SuperSecret@12345
```

Kali-də yoxla:
```bash
echo "54321@terceSrepuS" | rev
# SuperSecret@12345
```

---

## 6. ✅ Credentials

```
Username: h3ck3rBoi
Password: SuperSecret@12345
```

---

## 7. 🚩 Flag Almaq

### Metod 1 — Login səhifəsindən:
```
http://10.82.167.31/login.html
Username: h3ck3rBoi
Password: SuperSecret@12345
→ Login düyməsinə bas → Flag görünür!
```

### Metod 2 — Birbaşa URL ilə:

Source kodda flag faylının yolu görünürdü:
```javascript
xhttp.open("GET", "RandomLo0o0o0o0o0o0o0o0o0o0gpath12345_Flag_"+a.value+"_"+b.value+".txt", true);
```

Yəni flag faylı:
```
http://10.82.167.31/RandomLo0o0o0o0o0o0o0o0o0o0gpath12345_Flag_h3ck3rBoi_SuperSecret@12345.txt
```

Birbaşa bu URL-i açsaq flag əldə edirik! 🚩

---

## 8. 🗺️ Attack Chain

```
Nmap → Port 80 (Apache)
    ↓
http://10.82.167.31/login.html
    ↓
Ctrl+U → Source kod baxıldı
    ↓
JavaScript-də şifrə tapıldı
    ↓
"54321@terceSrepuS" → tərsinə → SuperSecret@12345
    ↓
h3ck3rBoi : SuperSecret@12345 ilə login
    ↓
FLAG 🚩
```

---

## 9. 📝 Əsas Öyrənilənlər

| Konsept | İzah |
|---------|------|
| **Client-side auth** | Şifrə yoxlaması brauzerdə edilir — təhlükəlidir! |
| **Source code review** | Ctrl+U ilə HTML/JS kodunu görmək mümkündür |
| **RevereString** | Stringi tərsinə çevirərək şifrəni gizlətməyə cəhd |
| **Directory listing** | `/assets/` açıq idi — bütün fayllar görünürdü |
| **Hardcoded credentials** | Şifrə kodun içinə yazılmışdı |

---

## 10. ⚠️ Mühüm Qeydlər

> **Client-side authentication heç vaxt etibarlı deyil!**
> JavaScript kodu brauzerdə işləyir — hər kəs görə bilər.
> Şifrə yoxlaması həmişə **server tərəfində** edilməlidir!

> **Şifrəni gizlətmək onu qorumur!**
> `RevereString` sadəcə şifrəni tərsinə yazır — bu şifrələmə deyil!
> Həqiqi şifrələmə üçün bcrypt, argon2 kimi alqoritmlər lazımdır.

---

*Hazırladı: CTF Player | TryHackMe — CyberHeroes*
