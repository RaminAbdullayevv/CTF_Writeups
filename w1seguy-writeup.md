# TryHackMe — W1seGuy CTF Writeup (Azərbaycan dilində)

**Çətinlik:** Asan  
**Kateqoriya:** Kriptoqrafiya  
**Link:** https://tryhackme.com/room/w1seguy

---

## Giriş

W1seGuy — XOR şifrələməsini öyrənmək üçün əla bir CTF-dir. Bu tapşırıqda bizə şifrəli mətn verilir və biz **Known Plaintext Attack** (Məlum Açıq Mətn Hücumu) istifadə edərək şifrələmə açarını tapmalıyıq.

---

## Tapşırıq 1 — Mənbə Kodu

Əvvəlcə verilən Python mənbə kodunu oxuyuruq:

```python
def start(server):
    res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
    key = str(res)
    hex_encoded = setup(server, key)
    send_message(server, "This XOR encoded text has flag 1: " + hex_encoded + "\n")
    
    send_message(server,"What is the encryption key? ")
    key_answer = server.recv(4096).decode().strip()

    if key_answer == key:
        send_message(server, "Congrats! That is the correct key! Here is flag 2: " + flag + "\n")
```

### Koddan Nə Anladıq?

1. Server **5 simvollu** random açar yaradır
2. Açar `ascii_letters + digits` — yəni hərflər və rəqəmlər
3. Flag bu açarla **XOR** şifrələnir
4. Hex formatında bizə göndərilir
5. Biz açarı tapıb göndərməliyik
6. Düzgünsə — **Flag 2** verilir

---

## Tapşırıq 2 — Həll

### XOR-un Xüsusiyyəti

XOR simmetrikdir:
```
Flag XOR Açar = Şifrəli
Şifrəli XOR Flag = Açar
Şifrəli XOR Açar = Flag
```

### Known Plaintext Attack

Biz bilirik ki:
- Hər TryHackMe flagı **`THM{`** ilə başlayır
- Hər flag **`}`** ilə bitir

Bu 5 simvol bizə **tam 5 simvollu açarı** tapmağa kömək edir!

```
Şifrəli[0] XOR 'T' = Açar[0]
Şifrəli[1] XOR 'H' = Açar[1]
Şifrəli[2] XOR 'M' = Açar[2]
Şifrəli[3] XOR '{' = Açar[3]
Şifrəli[-1] XOR '}' = Açar[4]
```

### Avtomatik Script

```python
import socket

HOST = "10.81.159.236"
PORT = 1337

s = socket.socket()
s.connect((HOST, PORT))

data = s.recv(1024).decode()
print(data)

hex_encoded = data.split(": ")[1].strip()
encrypted = bytes.fromhex(hex_encoded)

# THM{ ilə başlayır, } ilə bitir
known = b"THM{"
key_start = []
for i in range(4):
    key_start.append(encrypted[i] ^ known[i])

# Son simvolu } ilə tap
last_byte = encrypted[-1] ^ ord("}")
key = bytes(key_start) + bytes([last_byte])

print(f"Açar: {key.decode()}")

# Flagı decode et
result = ""
for i, byte in enumerate(encrypted):
    result += chr(byte ^ key[i % len(key)])
print(f"Flag 1: {result}")

# Açarı servera göndər
s.recv(1024)
s.send((key.decode() + "\n").encode())
response = s.recv(1024).decode()
print(f"Flag 2: {response}")
s.close()
```

### Nəticə

```
Açar:   OpFCM
Flag 1: THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}
Flag 2: THM{BrUt3_ForC1nG_XOR_cAn_B3_FuN_nO?}
```

---

## Öyrəndiklərimiz

| Mövzu | İzah |
|-------|------|
| XOR şifrələmə | Simmetrik, eyni açarla şifrələyir və açır |
| Known Plaintext Attack | Açıq mətni bilsən açarı tapa bilərsən |
| Qısa açar zəifliyi | 5 simvollu açar çox asanlıqla tapılır |
| Python socket | Serverlə avtomatik əlaqə qurmaq |

---

## Defterə Qeyd

> **XOR Known Plaintext Attack** — şifrəli mətnin bir hissəsini bilirsənsə, XOR-un simmetrik xüsusiyyətindən istifadə edərək açarı tapa bilərsən.  
> Düstur: `Şifrəli XOR Açıq_Mətn = Açar`

---

*Writeup: TryHackMe W1seGuy CTF — Azərbaycan dilində*
