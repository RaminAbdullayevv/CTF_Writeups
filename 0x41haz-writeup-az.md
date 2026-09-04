# TryHackMe — 0x41haz CTF Writeup (Azərbaycan dilində)

**Çətinlik:** Asan  
**Kateqoriya:** Reverse Engineering  
**Platforma:** TryHackMe  
**Keçid:** https://tryhackme.com/room/0x41haz

---

## Tapşırıq nə deyir?

> "Bu sadə bir reversing çəllengidir. Binary faylı yüklə və analiz et, passwordu tap. Anti-reversing tədbirləri ola bilər!"

---

## İstifadə olunan alətlər

| Alət | Məqsəd |
|------|--------|
| `file` | Faylın növünü öyrənmək |
| `xxd` | Hex dump — faylın içini görmək |
| `strings` | İçindəki mətn parçalarını tapmaq |
| `dd` | ELF header-i düzəltmək |
| `ltrace` | Proqramın çağırdığı funksiyaları izləmək |
| `gdb` | Debugger — proqramı addım-addım izləmək |

---

## Addım 1 — Faylı tanı

```bash
file 0x41haz-1640335532346.0x41haz
```

**Nəticə:**
```
0x41haz: ELF 64-bit MSB *unknown arch 0x3e00* (SYSV)
```

### Problem nədir?

- **MSB (Big-Endian)** yazır — amma x86-64 Linux **LSB (Little-Endian)** istifadə edir
- Bu **qəsdən edilmiş** bir anti-reversing tədbirdir
- `gdb`, `ltrace`, `objdump` bu faylı tanımır

---

## Addım 2 — Strings ilə analiz

```bash
strings 0x41haz-1640335532346.0x41haz
```

**Maraqlı nəticələr:**
```
Hey , Can You Crackme ?
Tell Me the Password :
Is it correct , I don't think so.
Well Done !!
2@@25$gfH       ← şübhəli string!
```

`2@@25$gfH` şifrələnmiş password kimi görünür, amma bu `strings`-in birbaşa tapdığı şeydir — həqiqi passwordu hələ tapmamışıq.

---

## Addım 3 — Faylı işlət

```bash
chmod +x 0x41haz-1640335532346.0x41haz
./0x41haz-1640335532346.0x41haz
```

**Nəticə:**
```
=======================
Hey , Can You Crackme ?
=======================
It's jus a simple binary
Tell Me the Password :
```

Proqram **password soruşur**. Biz isə həmin passwordu tapmaq üçün analiz edirik.

---

## Addım 4 — ELF Header-i düzəlt (Anti-reversing həlli)

### Niyə lazımdır?

ELF faylının **6-cı byte-ı** (offset `0x05`) endianness-i göstərir:

```
0x01 = LSB (Little-Endian) ← düzgün olan
0x02 = MSB (Big-Endian)    ← faylda bu var (yanlış)
```

`xxd` ilə yoxlayaq:
```bash
xxd 0x41haz-1640335532346.0x41haz | head -5
```

```
00000000: 7f45 4c46 02 02 01 00 ...
                      ↑
                   offset 0x05 = 02 (MSB — yanlış!)
```

### Düzəltmək üçün:

```bash
printf '\x01' | dd of=0x41haz-1640335532346.0x41haz bs=1 seek=5 conv=notrunc
```

### Yoxla:

```bash
file 0x41haz-1640335532346.0x41haz
```

```
0x41haz: ELF 64-bit LSB pie executable, x86-64 ✅
```

İndi fayl düzgün tanınır!

---

## Addım 5 — GDB ilə Debug

```bash
gdb ./0x41haz-1640335532346.0x41haz
```

Entry point-i tap:
```
(gdb) info file
Entry point: 0x1080
```

Breakpoint qoy və işlət:
```
(gdb) break *0x1080
(gdb) run
(gdb) layout asm
(gdb) nexti   ← addım-addım irəlilə
```

Password müqayisəsi zamanı **RAX register**-ini yoxla:
```
(gdb) info registers rax
```

RAX-da **həqiqi password** görünür!

---

## Addım 6 — ltrace ilə yoxla

```bash
ltrace ./0x41haz-1640335532346.0x41haz
```

ltrace proqramın çağırdığı funksiyaları göstərir:
```
puts("Tell Me the Password :")
gets(...)
strlen(...)
```

Password tapılandan sonra yoxla:
```bash
echo "TAPILAN_PASSWORD" | ./0x41haz-1640335532346.0x41haz
```

```
Well Done !! ✅
```

---

## Öyrəndiklərimiz

| Mövzu | İzah |
|-------|------|
| **ELF Header** | Binary faylın metadata-sı — növ, arch, endianness saxlayır |
| **Endianness** | LSB vs MSB — byte-ların yaddaşda sıralanma qaydası |
| **Anti-reversing** | 1 byte dəyişdirərək alətləri çaşdırmaq |
| **strings** | Binary içindəki mətnləri tapmaq |
| **GDB** | Proqramı işləyərkən içindən izləmək |
| **ltrace** | Library funksiya çağırışlarını izləmək |

---

## Nəticə

Bu çəlleng **anti-reversing texnikasını** öyrədir:

```
1. Fayl işləyir          ✅ (proqram açılır)
2. Amma tools işləmir    ❌ (MSB header)
3. 1 byte dəyişdiririk   🔧
4. Tools işləyir         ✅
5. Password tapılır      🎯
```

> **Əsas dərs:** Binary faylın header-i korlanıbsa, analiz etməzdən əvvəl düzəlt!

---

*Writeup: TryHackMe 0x41haz çəllengi — Azərbaycan dilində*
