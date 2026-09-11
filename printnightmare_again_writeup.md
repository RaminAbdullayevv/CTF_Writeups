# TryHackMe — PrintNightmare, again! Writeup (Azərbaycan dilində)

**Otaq linki:** https://tryhackme.com/room/printnightmarec2bn7l  
**Çətinlik:** Orta  
**Mövzu:** Windows Print Spooler zəifliyi — Defensive/Forensics

---

## Ümumi Baxış

Bu lab-da biz bir işçinin **PrintNightmare (CVE-2021-1675)** exploit-indən istifadə edərək öz kompütərində səlahiyyətlərini artırıb-artırmadığını araşdırırıq. Rol **Blue Team (Müdafiə tərəfi)** — yəni biz detektiv kimi izləri axtarırıq.

---

## İstifadə olunan alət

**FullEventLogView** (NirSoft) — Windows Event Log-larını rahat oxumaq üçün pulsuz alət.

---

## Hazırlıq

1. Virtual maşını işə sal
2. **FullEventLogView.exe** faylını tap və aç
3. `Options > Advanced Options` → **"Show events from all times"** seçimini aktiv et → OK bas
4. Artıq bütün event log-lar görünür

---

## Sual 1: İstifadəçi hansı zip faylını yüklədi?

**Nə axtarırıq:** Downloads qovluğuna yüklənmiş `.zip` faylının adı

**Necə tapırıq:**
- **Ctrl+Q** ilə Quick Filter aç
- `.zip` yaz və Enter bas
- Description sütununda zip faylının adı görünəcək

**İz:** `downloads` qovluğunda saxlanılıb

---

## Sual 2: İstifadəçinin icra etdiyi exploit-in tam yolu nədir?

**Nə axtarırıq:** PowerShell exploit skriptinin tam fayl yolu

**Necə tapırıq:**
- Quick Filter-ə `CVE` yaz
- Nəticələrdə `C:\Users\bmurphy\Downloads\...` ilə başlayan `.ps1` faylını tap

**İz:** CVE-2021-1675 exploit-i PowerShell skripti kimi icra edilib

---

## Sual 3: Zərərli DLL-in müvəqqəti saxlandığı yer nədir?

**Nə axtarırıq:** Zərərli DLL faylının `Temp` qovluğundakı yolu

**Necə tapırıq:**
- Quick Filter-ə `.dll` yaz
- Description sütununda `C:\Windows\Temp\...` ilə başlayan yolu tap

**İz:** DLL əvvəlcə Temp qovluğuna endirilir, sonra yüklənir

---

## Sual 4: DLL-in yükləndiyə tam yol nədir?

**Nə axtarırıq:** DLL-in əsl yükləmə yolu (Temp-dən fərqli ola bilər)

**Necə tapırıq:**
- `.dll` axtarışını davam etdir
- Timeline-da aşağı scroll edərək DLL-in yükləmə yolunu tap

---

## Sual 5: Bu hücumla əlaqəli əsas registry yolu nədir?

**Nə axtarırıq:** PrintNightmare-in istifadə etdiyi Windows Registry yolu

**Necə tapırıq:**
- Quick Filter-ə `HKLM` yaz
- Print Spooler ilə bağlı registry yolunu tap

**İz:** `HKLM\...\CurrentControlSet\Control\Print` kimi bir yol

---

## Sual 6: Microsoft imzalı olmayan binary-ni yükləməkdən bloklanacaq prosesin PID-i nədir?

**Nə axtarırıq:** Bloklanmış prosesin Process ID nömrəsi

**Necə tapırıq:**
- Quick Filter-ə `blocked` yaz
- İkinci sətirdəki **Blocked PID** dəyərini tap
- Həmçinin `Process ID` sütununa bax

---

## Sual 7: Yeni yaradılmış lokal administrator hesabının username-i nədir?

**Nə axtarırıq:** Exploit vasitəsilə yaradılmış yeni Windows hesabının adı

**Necə tapırıq:**
- Quick Filter-ə `4720` yaz (bu Event ID yeni user yaradıldığında qeydə alınır)
- Description-da yeni hesabın adını tap

---

## Sual 8: Bu istifadəçinin parolu nədir?

**Nə axtarırıq:** Yeni yaradılmış hesabın parolu

**Necə tapırıq:**
- Bu məlumat **PowerShell Console History** faylında saxlanılır
- Virtual maşında PowerShell aç və bu komandanı işlət:

```powershell
type C:\Users\bmurphy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

- `net user` komandaları arasında parolu tap

---

## Sual 9: İstifadəçi izlərini silmək üçün hansı 2 komandanı icra etdi?

**Nə axtarırıq:** Log-ları silmək üçün istifadə edilmiş 2 komanda (vergüllə, boşluqsuz)

**Necə tapırıq:**
- Yuxarıdakı eyni PowerShell history faylına bax:

```powershell
type C:\Users\bmurphy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

- `wevtutil` ilə başlayan komandaları tap
- Cavab formatı: `komanda1,komanda2`

**İz:** `wevtutil cl` komadasıyla log-lar silinir

---

## Ümumi Nəticə

Bu lab-da öyrəndiklərimiz:

| Mərhələ | Nə etdi hacker | Biz nə tapdıq |
|---------|---------------|----------------|
| 1 | ZIP faylı yüklədi | FullEventLogView → `.zip` axtarışı |
| 2 | PowerShell exploit işlətdi | FullEventLogView → `CVE` axtarışı |
| 3 | Zərərli DLL yaratdı | FullEventLogView → `.dll` axtarışı |
| 4 | Registry-ə yazdı | FullEventLogView → `HKLM` axtarışı |
| 5 | Yeni admin user yaratdı | Event ID `4720` |
| 6 | İzlərini sildi | PowerShell history faylı |

---

## PrintNightmare haqqında qısa məlumat

- **CVE:** CVE-2021-1675 / CVE-2021-34527
- **Zəiflik:** Windows Print Spooler servisi
- **Nəticə:** İstənilən authenticated user SYSTEM səlahiyyəti qazana bilir
- **Patch:** Microsoft 2021-ci ildə yamaq buraxdı

---

*Writeup Azərbaycan dilində hazırlanmışdır.*
