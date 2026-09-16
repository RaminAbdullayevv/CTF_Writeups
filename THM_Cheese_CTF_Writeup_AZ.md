# TryHackMe — Cheese CTF Write-up

> Bu write-up yalnız TryHackMe laboratoriyası üçündür. `TARGET_IP` öz maşının IP-si ilə əvəz et.

## Məqsəd

Maşına daxil olmaq, `user.txt` və `root.txt` flag-lərini tapmaq.

## 1. Nmap scan

```bash
nmap -sC -sV -p- -T4 TARGET_IP -oN nmap.txt
```

Əvvəlcə açıq portları və xidmətləri müəyyən et. HTTP portu web tətbiqinə, SSH isə sonrakı girişə görə vacibdir. Nəticədə əlavə yüksək port görsən, onu da ayrıca yoxla.

## 2. Web enumeration

```bash
gobuster dir -u http://TARGET_IP/ \
 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

Ana səhifədə və tapılan qovluqlarda source code-u, comment-ləri və upload funksiyalarını yoxla. Bu otaqda əsas maraqlı hissə fayl yükləmə funksiyasıdır.

## 3. Upload funksiyasının yoxlanması

Adi şəkil faylı yüklə və cavabı Burp Suite və ya browser developer tools ilə yoxla. Faylın serverə hansı adla və hansı qovluğa yazıldığını müəyyən et.

Əgər server PHP fayllarını işlədirdisə, test məqsədilə yalnız THM maşınında PHP faylı ilə yoxlama aparılır. Upload filter alternativ uzantıları bloklamaya bilər. Məsələn, `.php` əvəzinə tətbiqin qəbul etdiyi başqa PHP uzantısı istifadə olunur.

## 4. İlkin shell

Kali-də listener aç:

```bash
nc -lvnp 4444
```

Reverse shell faylında `ATTACKER_IP` yerinə Kali IP-ni yaz, faylı upload et və upload olunmuş faylın URL-sini aç. Shell gəldikdən sonra:

```bash
whoami
id
pwd
ls -la
```

Əgər shell zəifdirsə, onu interaktiv et:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

## 5. User flag

İstifadəçiləri yoxla:

```bash
cat /etc/passwd | grep -E 'bash|sh$'
find / -name user.txt 2>/dev/null
```

Tapılan flag-i oxu:

```bash
cat /home/USER/user.txt
```

## 6. Sistem enumeration

İndi məqsəd root-a keçməkdir. Aşağıdakı əsas əmrləri işlət:

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
ss -lntup
ps aux
```

Xüsusilə bunlara bax:

- `sudo -l` nəticəsində parolsuz icazə verilən komandalar;
- SUID binary-lər;
- lokalda işləyən, lakin Nmap-də görünməyən servislər;
- yazıla bilən script və config faylları;
- cron job-lar.

## 7. Privilege escalation

Əgər `sudo -l` konkret bir proqram göstərirsə, həmin proqramın GTFOBins-də privilege escalation imkanı olub-olmadığını yoxla. Məsələn, icazəli proqram shell açmağa imkan verirsə, onu yalnız laboratoriyada istifadə et.

Yazıla bilən faylları tap:

```bash
find /opt /usr/local /var/www -type f -writable 2>/dev/null
```

Cron proseslərini görmək üçün:

```bash
cat /etc/crontab
ls -la /etc/cron.*
```

Əgər root tərəfindən işlədilən script adi istifadəçi tərəfindən dəyişdirilə bilirsə, script-in zəifliyindən istifadə etməklə root shell əldə etmək mümkündür.

## 8. Root flag

Root shell əldə etdikdən sonra:

```bash
whoami
id
cat /root/root.txt
```

## Qısa yekun

1. Nmap ilə açıq portları tap.
2. Gobuster ilə gizli qovluqları araşdır.
3. Upload funksiyasının filterini yoxla.
4. Upload zəifliyi ilə ilkin shell əldə et.
5. `user.txt` tap.
6. `sudo -l`, SUID, cron və yazıla bilən faylları yoxla.
7. Misconfiguration-dan istifadə edib root ol.
8. `/root/root.txt` faylını oxu.

## Öyrənilən mövzular

- Web directory enumeration
- File upload zəiflikləri
- Reverse shell
- Linux istifadəçi və icazə enumeration-u
- SUID və sudo misconfiguration
- Cron və writable script privilege escalation
