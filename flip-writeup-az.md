# TryHackMe — Flip | Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** [Flip](https://tryhackme.com/room/flip)  
**Çətinlik:** Asan  
**Kateqoriya:** Kriptoqrafiya, AES CBC Bit Flipping  
**Müəllif:** ramin  

---

## 📋 Ümumi Baxış

Bu CTF-də AES CBC şifrələməsinin zəifliyindən istifadə edərək autentifikasiyanı bypass etmək lazımdır. Server port 1337-də TCP dinləyir.

---

## 🔍 Mərhələ 1 — Mənbə Kodunu Analiz etmək

Task faylında Python kodu verilir. Əsas məqamlar:

```python
# Server yoxlayır:
if b'admin&password=sUp3rPaSs1' in unpad(paddedParams,16,style='pkcs7'):
    return 1  # Giriş uğurlu!

# Birbaşa admin yazmaq qadağandır:
if "admin&password=sUp3rPaSs1" in message:
    send_message(server, 'Not that easy :)')
```

**Nə başa düşdük:**
- Şifrə `sUp3rPaSs1`-dir
- Admin adını birbaşa yaza bilmirik
- Server bizə şifrəli mətni göstərir — bu zəiflikdir!

---

## 🧠 Mərhələ 2 — CBC Bit Flipping nədir?

AES CBC şifrələməsində hər blok bir-birinə bağlıdır:

```
Şifrələmə:
Plaintext  → XOR (əvvəlki CT bloku ilə) → AES encrypt → Ciphertext

Açma:
Ciphertext → AES decrypt → XOR (əvvəlki CT bloku ilə) → Plaintext
```

**Vacib xüsusiyyət:**
Əgər şifrəli mətndə bir baytı dəyişsən — açıldıqda **həmin baytın plaintext-dəki müvafiq baytı** dəyişir!

```
CT[0] dəyişdirilərsə → Plaintext[16] dəyişir (blok 2-nin əvvəli)
```

---

## 💥 Mərhələ 3 — Hücum

### Addım 1 — Qoşulun

```bash
nc MACHINE_IP 1337
```

### Addım 2 — Credentials göndərin

```
Username: bdmin
Password: sUp3rPaSs1
```

**Niyə `bdmin`?**
```
"access_username=bdmin&password=sUp3rPaSs1"
                 ↑
                 b → a flip edəcəyik
                 Server açanda "admin" görəcək!
```

### Addım 3 — Leaked Ciphertext gəlir

```
Leaked ciphertext: 6cad6aafb4d8e434c6f1a67be6eeabea...
```

### Addım 4 — Birinci baytı flip edin

```bash
python3 << 'EOF'
from binascii import unhexlify, hexlify

ct = "LEAKED_CT_BURAYA"
ct_bytes = bytearray(unhexlify(ct))
ct_bytes[0] ^= ord('b') ^ ord('a')
print(hexlify(ct_bytes).decode())
EOF
```

**Niyə `ct_bytes[0]`?**
```
Plaintext: "access_username=bdmin..."
                            ↑
                         indeks 16 (blok 2-nin başlanğıcı)

CBC qaydası:
Blok 2-dəki dəyişiklik üçün → Blok 1-i dəyişirik
Blok 1 = CT-nin 0-15-ci baytları
b blok 2-nin 0-cı baytıdır → CT-nin 0-cı baytını dəyişirik!
```

### Addım 5 — Modified CT-ni göndərin

`enter ciphertext:` soruşanda çıxanı yapışdırın:

```
enter ciphertext: 6fad6aafb4d8e434...
```

### Nəticə:

```
No way! You got it!
A nice flag for you: THM{FliP_DaT_B1t_oR_G3t_Fl1pP3d}
```

---

## 🗺️ Bütün Prosesin Xəritəsi

```
🎯 Hədəf: Port 1337
        ↓
📄 Mənbə kodu oxundu
   → admin&password=sUp3rPaSs1 lazımdır
   → Birbaşa yazmaq qadağandır
        ↓
🔐 "bdmin" + "sUp3rPaSs1" göndərildi
        ↓
💡 Leaked ciphertext alındı
        ↓
🔄 CT[0] ^= ord('b') ^ ord('a')
   b → a flip edildi
        ↓
📤 Modified CT göndərildi
        ↓
🚩 FLAG: THM{FliP_DaT_B1t_oR_G3t_Fl1pP3d}
```

---

## 🧠 Texniki İzah — Niyə İşlədi?

CBC açma prosesi:

```
Plaintext[i] = AES_Decrypt(CT[i]) XOR CT[i-1]
```

Biz `CT[0]`-ı dəyişdik:
```
Köhnə: CT[0] = 6c
Yeni:  CT[0] = 6f  (6c XOR ord('b') XOR ord('a'))

Açma zamanı:
Plaintext[16] = AES_Decrypt(CT[1]) XOR CT[0]
              = ... XOR 6f
              = 'a'  ← 'b' əvəzinə!
```

Nəticə: `bdmin` → `admin` 🎯

---

## ⚠️ Yan effekt

CBC Bit Flipping-də bir bloku dəyişdikdə **həmin blok tamamilə pozulur**:

```
Blok 1 dəyişdirildikdə:
- Blok 2-nin 0-cı baytı düzəlir ✅
- Blok 1-in açılması tamamilə korlanır ❌
```

Amma bizim halda server yalnız `admin&password=sUp3rPaSs1` olub-olmadığını yoxlayır — blok 1-in pozulması əhəmiyyətsizdir!

---

## 📚 Öyrənilən Konsepsiyalar

| Konsepsiya | İzah |
|-----------|------|
| **AES CBC** | Hər blok əvvəlki blokla XOR-lanır |
| **Bit Flipping** | CT-dəki dəyişiklik plaintext-ə təsir edir |
| **Məlum Plaintext** | Biz mətni bildiyimiz üçün dəqiq flip edə bilirik |
| **Integrity** | Şifrələmə + MAC olmalıdır, tək şifrələmə kifayət deyil |

---

## 🛡️ Müdafiə Üsulları

1. **HMAC** — şifrəli mətni imzalamaq
2. **AEAD** — AES-GCM kimi integrity + şifrələmə birlikdə
3. **Ciphertext-i trust etməmək** — həmişə integrity yoxlamaq

---

## 🏁 Nəticə

Bu CTF-in əsas dərsi: **Şifrələmə ≠ Güvənlik**. AES CBC güclü şifrələmə olsa da, integrity yoxlaması olmadan bit flipping hücumuna açıqdır. Müasir sistemlər AES-GCM kimi AEAD alqoritmlərindən istifadə etməlidir.

**Flag:** `THM{FliP_DaT_B1t_oR_G3t_Fl1pP3d}` ✅

---

*Writeup: TryHackMe Flip otağı üçün Azərbaycan dilində hazırlanmışdır.*
