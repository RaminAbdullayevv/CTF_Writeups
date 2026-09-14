# mKingdom — TryHackMe WriteUp

## Məlumat

| | |
|---|---|
| **Platforma** | TryHackMe |
| **Maşın** | mKingdom |
| **Çətinlik** | Easy/Medium |
| **OS** | Linux (Ubuntu 14.04) |
| **IP** | 10.81.158.49 |

---

## Xülasə

```
Nmap → Concrete5 8.5.2 → admin:password → 
PHP Upload → RCE → www-data → toad → mario → 
pspy → Cron Job → /etc/hosts hijack → ROOT
```

---

## Mərhələlər

### 1. Reconnaissance
- `nmap -sV -p- 10.81.158.49`
- Yalnız **port 85** açıq — Concrete5 8.5.2

### 2. Web Enumeration
- Blog səhifəsindən `admin` istifadəçi adı tapıldı
- CMS versiyası: `meta generator` taqından
- Conversation tokenləri, fayl ID-ləri aşkar edildi

### 3. Initial Access
- Login: `admin:password`
- PHP upload icazəsi aktivləşdirildi
- `shell.php` yükləndi
- Reverse shell: `www-data`

### 4. Lateral Movement
- `www-data` → `toad` (şifrə: `ikaTeNTANtES`)
- `env` dəyişənlərində `PWD_token` tapıldı
- Base64 decode: `mariodayam`
- `toad` → `mario`

### 5. Privilege Escalation
- `pspy` ilə root cron tapıldı:
  ```
  curl mkingdom.thm:85/.../counter.sh | bash
  ```
- `/etc/hosts` mario tərəfindən yazıla bilirdi
- DNS hijacking: `mkingdom.thm` → Kali IP
- Saxta `counter.sh` host edildi
- **ROOT shell əldə edildi!**

---

## Şifrələr

| İstifadəçi | Şifrə |
|---|---|
| admin (CMS) | password |
| toad | ikaTeNTANtES |
| mario | mariodayam |

---

## İstifadə olunan texnikalar

| # | Texnika | Fayl |
|---|---|---|
| 1 | Reconnaissance | `01_reconnaissance.md` |
| 2 | Web Enumeration | `02_web_enumeration.md` |
| 3 | Brute Force | `03_brute_force.md` |
| 4 | File Upload RCE | `04_file_upload.md` |
| 5 | Lateral Movement | `05_lateral_movement.md` |
| 6 | Privilege Escalation | `06_privilege_escalation.md` |

---

## Alınan dərslər

1. CMS versiyasını həmişə yoxla — köhnə versiyalarda CVE-lər var
2. Sadə şifrələri (`admin:password`) əvvəlcə sına
3. Environment dəyişənlərini (`env`) həmişə yoxla
4. `/etc/hosts` icazələrini yoxla — DNS hijacking üçün qızıl fürsətdir
5. `pspy` — root proseslərini görmək üçün mütləq istifadə et
6. Cron job-lar URL-dən skript çəkirsə — kritik zəiflikdir
