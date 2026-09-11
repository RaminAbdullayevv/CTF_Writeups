# TryHackMe — Surfer

## Otaq haqqında

**Surfer** TryHackMe-də əsasən **SSRF (Server-Side Request Forgery)** zəifliyini öyrədən orta səviyyəli Linux veb labıdır. Ssenaridə veb tətbiqi istifadəçidən URL qəbul edir və həmin URL-ə server tərəfindən sorğu göndərir.

> Bu sənəd yalnız TryHackMe laboratoriyası üçün hazırlanıb. Eyni üsulları icazəsiz sistemlərdə tətbiq etmək olmaz.

## Öyrənilən mövzular

- Nmap ilə xidmət aşkarlanması
- Directory enumeration
- SSRF zəifliyinin tanınması
- Lokal və daxili endpoint-lərin yoxlanması
- Veb tətbiqindən əldə edilən məlumatların qiymətləndirilməsi
- SSH ilə giriş
- Linux privilege escalation
- Sudo icazələrinin yoxlanması

## 1. Maşını başladın

TryHackMe otağında lab maşınını başladın və IP ünvanını qeyd edin:

```text
MACHINE_IP = 10.10.x.x
```

Öz Kali maşınınızdan və ya AttackBox-dan işləyin.

## 2. İlkin Nmap yoxlaması

Əvvəl açıq portları müəyyən edin:

```bash
nmap -sC -sV -p- -oN nmap.txt MACHINE_IP
```

Sonra nəticəni qısa formada yoxlayın:

```bash
cat nmap.txt
```

Əsas diqqət:

- HTTP/HTTPS portları;
- SSH portu;
- server versiyası;
- veb serverin redirect etdiyi hostname.

Əgər hostname göstərilirsə, onu `/etc/hosts` faylına əlavə edin:

```text
MACHINE_IP surfer.thm
```

## 3. Veb tətbiqini araşdırın

Brauzerdə aşağıdakı ünvanı açın:

```text
http://MACHINE_IP
```

Səhifədəki bütün linkləri, formaları və URL parametrlərini yoxlayın. Xüsusilə aşağıdakı tip funksiyalara diqqət edin:

- URL göndərib səhifənin məzmununu göstərən funksiya;
- link yoxlayıcı;
- image/page preview;
- fetch və ya proxy funksiyası.

Directory enumeration üçün:

```bash
gobuster dir -u http://MACHINE_IP -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

Burada məqsəd tətbiqin bütün endpoint-lərini tapmaqdır. Bir endpoint istifadəçidən URL alırsa, bu SSRF üçün əsas namizəddir.

## 4. SSRF nədir?

SSRF zamanı server istifadəçinin verdiyi URL-ə özü sorğu göndərir.

Normal halda:

```text
Sənin brauzerin → xarici sayt
```

SSRF zamanı:

```text
Sənin brauzerin → hədəf server → başqa resurs
```

Bu, serverin özünə və ya yalnız daxili şəbəkədən əlçatan olan xidmətlərə sorğu göndərməyə səbəb ola bilər.

## 5. SSRF parametrini müəyyən edin

Tətbiqdə URL qəbul edən parametr tapdıqda əvvəlcə normal, zərərsiz URL ilə davranışı yoxlayın. Məsələn, tətbiqdə parametr `url` adlanırsa, sorğu təxminən belə görünə bilər:

```text
http://MACHINE_IP/page?url=http://example.com
```

Parametrin adını və cavab formatını öz labındakı real sorğuya uyğunlaşdırın. Əsas müşahidələr:

- server xarici səhifənin məzmununu qaytarırmı;
- cavab vaxtı dəyişirmi;
- status kodu dəyişirmi;
- səhifə yalnız müəyyən URL-lərə icazə verirmi.

## 6. Lokal serverə sorğu yoxlayın

SSRF-in olub-olmadığını yoxlamaq üçün lab daxilində serverin özünə yönəlmiş URL-ləri test edin:

```text
http://127.0.0.1/
http://localhost/
```

Əgər tətbiqin cavabı dəyişir və daxili səhifə görünürsə, server tərəfindən edilən sorğu təsdiqlənmiş olur.

Sonra yalnız otağın kontekstində daxili portları yoxlamaq olar. Məsələn, HTTP xidmətləri üçün tez-tez istifadə olunan portlar:

```text
http://127.0.0.1:80/
http://127.0.0.1:8080/
http://127.0.0.1:8000/
```

Burada məqsəd port scan etmək deyil, tətbiqin daxili resursu server adından gətirib-gətirmədiyini anlamaqdır.

## 7. Daxili panel və məlumatları analiz edin

SSRF vasitəsilə daxili səhifə açılarsa, aşağıdakılara baxın:

- administrator paneli;
- debug məlumatları;
- konfiqurasiya səhifələri;
- istifadəçi adları;
- login formaları;
- daxili xidmətlərin endpoint-ləri.

Məlumatı birbaşa nəticə kimi qəbul etməyin. Hər credential və endpoint-i ayrıca yoxlayın. HTML source koduna da baxın:

```text
Ctrl + U
```

və ya:

```bash
curl -i http://MACHINE_IP/
```

## 8. İlkin giriş

Əgər tətbiq daxili səhifədə SSH credential və ya istifadəçi adı göstərirsə, yalnız lab IP-si üzərində SSH ilə yoxlayın:

```bash
ssh USER@MACHINE_IP
```

Parol soruşulduqda labdan əldə etdiyiniz məlumatı istifadə edin.

Girişdən sonra:

```bash
whoami
id
hostname
pwd
```

İlk flag-i tapmaq üçün öz istifadəçi qovluğunuzu yoxlayın:

```bash
ls -la
find /home -maxdepth 2 -type f -name 'user.txt' 2>/dev/null
```

## 9. Linux privilege escalation yoxlaması

Əvvəl sudo icazələrini yoxlayın:

```bash
sudo -l
```

Sonra istifadəçinin qruplarını və sistem məlumatlarını yoxlayın:

```bash
id
uname -a
find / -perm -4000 -type f 2>/dev/null
```

Əgər `sudo -l` konkret bir proqramı parolsuz icra etməyə icazə verirsə, həmin proqramın təhlükəsiz laboratoriya mühitində root-a necə təsir edə biləcəyini GTFOBins və ya proqramın manual səhifəsi ilə analiz edin.

Əsas prinsip budur:

```text
sudo -l → icazəli proqramı müəyyən et → həmin proqramın davranışını analiz et
```

## 10. Root flag

Root səviyyəsinə çatdıqdan sonra:

```bash
whoami
cat /root/root.txt
```

Əgər `Permission denied` alırsınızsa, əvvəlcə həqiqətən root olduğunuzu yoxlayın:

```bash
id
```

## Nəticə

Surfer labının əsas dərsi serverin istifadəçinin verdiyi URL-ə kor-koranə sorğu göndərməsinin təhlükəli olmasıdır. SSRF daxili panellərin, lokal xidmətlərin və yalnız server tərəfindən görülə bilən resursların ifşa olunmasına səbəb ola bilər.

## Müdafiə tədbirləri

- İstifadəçidən qəbul edilən URL-lər allowlist ilə məhdudlaşdırılmalıdır.
- `localhost`, `127.0.0.1`, private IP diapazonları və metadata ünvanları bloklanmalıdır.
- DNS resolve-dan sonra IP yenidən yoxlanmalıdır.
- Redirect-lər də ayrıca yoxlanmalıdır.
- Serverin daxili xidmətləri autentifikasiya ilə qorunmalıdır.
- Egress firewall qaydaları tətbiq edilməlidir.
- Cavablar istifadəçiyə tam ötürülməməlidir.

## Qısa yaddaş qeydi

```text
Nmap
  ↓
Veb endpoint-lərini tap
  ↓
URL qəbul edən funksiyanı müəyyən et
  ↓
SSRF-i localhost ilə təsdiqlə
  ↓
Daxili panel və məlumatları analiz et
  ↓
İlkin giriş əldə et
  ↓
sudo -l və digər Linux yoxlamaları
  ↓
Flag-ləri oxu
```

## Mənbələr

- [TryHackMe — Surfer](https://tryhackme.com/room/surfer)
- [Surfer writeup — System Weakness](https://systemweakness.com/)
- [Surfer — Pentest Everything](https://viperone.gitbook.io/pentest-everything/)
- [Surfer writeup — siunam](https://siunam321.github.io/)

