---
layout: post
title: "Wireshark – Cheatsheet (CTF & Forensics)"
date: 2025-03-28
categories: [cheatsheet, networking, ctf, forensics]
tags: [wireshark, tshark, pcap, network-analysis, ctf, forensics, blue-team]
description: "Komplexný Wireshark cheatsheet pre CTF, forenzn&uacute; anal&yacute;zu a penetračné testovanie. Display filtre, capture filtre, tshark one-liners, dekryptovanie TLS/WPA a typické CTF scenáre."
---

Komplexný referenčný list pre prácu s Wiresharkom pri CTF, forenznej analýze sietí a penetračnom testovaní.

## Obsah
{:.no_toc}

* TOC
{:toc}

---

## Základy rozhrania

```
Hlavné panely:
  Packet List    – zoznam zachytených paketov
  Packet Details – stromový rozklad hlavičiek (kliknuteľné)
  Packet Bytes   – surové bajty (hex + ASCII)

Stavový riadok:
  Zobrazuje počet paketov: celkovo / zobrazených / označených
```

---

## Spustenie & Zachytávanie

```bash
wireshark                        # GUI
wireshark -i eth0                # zachytávať na konkrétnom rozhraní
wireshark -r file.pcap           # otvoriť súbor
wireshark -k -i eth0             # okamžite začať zachytávať

# CLI alternatíva (headless)
tshark -i eth0                   # zachytávať v termináli
tshark -r file.pcap              # čítať pcap
tshark -r file.pcap -Y "http"    # s display filtrom
tshark -i eth0 -w out.pcap       # zachytávať a uložiť
tshark -r file.pcap -T fields \
  -e ip.src -e ip.dst -e tcp.port  # exportovať konkrétne polia
```

---

## Capture Filtre (BPF syntax)

> Aplikujú sa **pred** zachytávaním – ovplyvňujú čo sa uloží.
> Iná syntax ako display filtre!

```
# Protokoly
tcp
udp
icmp
arp
dns
http

# Porty
port 80
port 443
port 22
not port 53
portrange 1-1024

# IP adresy
host 192.168.1.1
src host 10.0.0.1
dst host 10.0.0.1
net 192.168.1.0/24
src net 10.0.0.0/8

# Kombinácie
tcp and port 80
host 192.168.1.1 and not port 22
src host 10.0.0.1 and dst port 443
tcp or udp
not arp and not icmp

# Veľkosť paketu
less 128
greater 512
len == 0

# Flagy
tcp[13] & 0x02 != 0              # SYN pakety
tcp[13] & 0x01 != 0              # FIN pakety
tcp[13] & 0x04 != 0              # RST pakety
tcp[13] == 0x18                  # PSH+ACK
```

---

## Display Filtre

> Aplikujú sa **po** zachytávaní – iba filtrujú zobrazenie.

### Základná syntax

```
# Operátory
==    eq       rovná sa
!=    ne       nerovná sa
>     gt       väčší ako
<     lt       menší ako
>=    ge       väčší alebo rovný
<=    le       menší alebo rovný
&&    and      logické AND
||    or       logické OR
!     not      negácia
~     matches  regex (PCRE)
contains       obsahuje reťazec
```

### IP & Sieť

```
ip.addr == 192.168.1.1           # zdroj ALEBO cieľ
ip.src == 192.168.1.1            # iba zdroj
ip.dst == 192.168.1.1            # iba cieľ
ip.addr == 192.168.1.0/24        # celá sieť
ip.ttl < 10                      # nízke TTL
ip.ttl == 64
ip.proto == 6                    # TCP (6=TCP, 17=UDP, 1=ICMP)
!ip.addr == 192.168.1.1          # vylúčiť IP
ip.flags.mf == 1                 # fragmentované pakety
```

### TCP

```
tcp.port == 80
tcp.srcport == 443
tcp.dstport == 22
tcp.flags.syn == 1               # SYN
tcp.flags.fin == 1               # FIN
tcp.flags.rst == 1               # RST
tcp.flags.ack == 1               # ACK
tcp.flags.push == 1              # PSH
tcp.flags == 0x002               # iba SYN (bez ACK)
tcp.flags == 0x012               # SYN+ACK
tcp.analysis.retransmission      # retransmisie
tcp.analysis.duplicate_ack       # duplicate ACK
tcp.analysis.zero_window         # zero window
tcp.len > 0                      # pakety s dátami (nie iba ACK)
tcp.stream == 0                  # sledovať konkrétny stream
```

### UDP

```
udp.port == 53
udp.srcport == 4444
udp.length > 100
```

### HTTP

```
http                             # všetka HTTP prevádzka
http.request                     # iba requesty
http.response                    # iba odpovede
http.request.method == "GET"
http.request.method == "POST"
http.request.uri contains "login"
http.request.uri matches "\.php$"
http.response.code == 200
http.response.code == 404
http.response.code >= 400        # chybové odpovede
http.host == "example.com"
http.host contains "google"
http.user_agent contains "Mozilla"
http.cookie contains "session"
http.content_type contains "json"
http.authorization               # pakety s autorizáciou
```

### HTTPS / TLS

```
tls                              # všetka TLS prevádzka
tls.handshake                    # TLS handshake pakety
tls.handshake.type == 1          # Client Hello
tls.handshake.type == 2          # Server Hello
tls.record.version == 0x0303     # TLS 1.2
ssl.handshake.extensions_server_name contains "example.com"  # SNI
```

### DNS

```
dns                              # všetka DNS prevádzka
dns.qry.name == "example.com"
dns.qry.name contains "evil"
dns.qry.type == 1                # A záznam
dns.qry.type == 28               # AAAA záznam
dns.qry.type == 15               # MX záznam
dns.qry.type == 16               # TXT záznam
dns.flags.response == 0          # iba DNS requesty
dns.flags.response == 1          # iba DNS odpovede
dns.resp.len > 512               # veľké odpovede (DNS amplification)
dns.count.answers > 10           # podozrivo veľa odpovedí
```

### ICMP

```
icmp                             # všetko ICMP
icmp.type == 8                   # Echo Request (ping)
icmp.type == 0                   # Echo Reply
icmp.type == 3                   # Destination Unreachable
icmp.type == 11                  # Time Exceeded (traceroute)
icmp.data_len > 100              # veľké ICMP (možný tunel/exfil)
```

### ARP

```
arp                              # všetok ARP
arp.opcode == 1                  # ARP Request
arp.opcode == 2                  # ARP Reply
arp.src.hw_mac == aa:bb:cc:dd:ee:ff
arp.duplicate-address-detected   # ARP spoofing detekcia
```

### DHCP / BOOTP

```
dhcp                             # DHCP prevádzka
bootp.option.dhcp == 1           # DHCP Discover
bootp.option.dhcp == 2           # DHCP Offer
bootp.option.dhcp == 3           # DHCP Request
bootp.option.dhcp == 5           # DHCP ACK
bootp.hw.mac_addr == aa:bb:cc:dd:ee:ff
```

### FTP

```
ftp                              # FTP príkazy
ftp.request.command == "USER"
ftp.request.command == "PASS"
ftp.request.command == "RETR"    # stiahnutie súboru
ftp.request.command == "STOR"    # nahratie súboru
ftp-data                         # FTP dátový prenos
ftp.response.code == 230         # úspešné prihlásenie
```

### SMB

```
smb                              # SMB v1
smb2                             # SMB v2/3
smb.cmd == 0x72                  # Negotiate Protocol
smb2.cmd == 1                    # Session Setup
smb2.filename contains "password"
smb2.tree contains "C$"          # admin share
```

### SMTP / E-mail

```
smtp                             # SMTP
smtp.req.command == "AUTH"
smtp.req.command == "MAIL"
smtp.req.command == "RCPT"
smtp.req.command == "DATA"
imap                             # IMAP
pop                              # POP3
```

---

## Užitočné kombinované filtre pre CTF & Forensiku

```
# Prihlásenia cez HTTP (POST na login stranky)
http.request.method == "POST" and http.request.uri contains "login"

# Exfiltrácia cez DNS (dlhé subdomény)
dns and dns.qry.name matches "[a-f0-9]{20,}"

# Exfiltrácia cez ICMP (veľké pakety)
icmp.type == 8 and data.len > 100

# Nmap SYN scan
tcp.flags == 0x002 and not tcp.flags.ack == 1

# Port scan (veľa RST)
tcp.flags.rst == 1

# Reverse shell / netcat
tcp.dstport == 4444 or tcp.srcport == 4444

# Podozrivé User-Agenty
http.user_agent contains "curl"
http.user_agent contains "python"
http.user_agent contains "sqlmap"
http.user_agent contains "nmap"

# SQL injection pokusy
http.request.uri contains "UNION"
http.request.uri contains "SELECT"
http.request.uri contains "--"
http.request.uri contains "1=1"

# Credentials v čistom texte (HTTP Basic Auth)
http.authorization contains "Basic"

# Cleartext FTP heslá
ftp.request.command == "PASS"

# Telnet (cleartext session)
telnet

# Cleartext SMTP autentifikácia
smtp.req.command == "AUTH"

# Skenovanie (veľa SYN na rôzne porty)
tcp.flags.syn == 1 and tcp.flags.ack == 0

# ARP spoofing
arp.opcode == 2 and arp.duplicate-address-detected

# DNS tunneling (podozrivo veľa DNS)
dns and frame.len > 200

# Podozrivé destinácie (mimo bežných portov)
tcp and not (tcp.port == 80 or tcp.port == 443 or tcp.port == 22 or tcp.port == 53)

# Hľadanie flagov v tele HTTP odpovede (CTF)
http.response and frame contains "flag{"
http.response and frame contains "CTF{"
```

---

## Sledovanie Streamov

```
# GUI:
Pravý klik na paket → Follow → TCP Stream
Pravý klik na paket → Follow → UDP Stream
Pravý klik na paket → Follow → HTTP Stream
Pravý klik na paket → Follow → TLS Stream

# Filter pre konkrétny stream (číslo z Follow dialógu):
tcp.stream == 0
tcp.stream == 5
udp.stream == 2

# Zobrazenie: Show data as → ASCII / HEX / Raw
```

---

## Exportovanie dát

### Exportovanie súborov z prevádzky

```
# GUI: File → Export Objects
  HTTP    – extrahuje súbory prenesené cez HTTP (obrázky, exe, html...)
  SMB     – súbory z SMB prenosu
  DICOM   – medicínske súbory
  IMF     – e-mailové správy
  TFTP    – TFTP prenosy

# tshark – export HTTP objektov
tshark -r file.pcap --export-objects http,./output_dir/
tshark -r file.pcap --export-objects smb,./output_dir/
```

### Exportovanie paketov

```bash
# GUI: File → Export Specified Packets
# Uložiť len zobrazené (filtrované) pakety

# CLI
tshark -r input.pcap -Y "http" -w filtered.pcap    # filtrovať + uložiť
tshark -r file.pcap -Y "tcp.stream == 3" -w stream3.pcap
editcap -r file.pcap out.pcap 1-100                # pakety 1 až 100
mergecap -w merged.pcap file1.pcap file2.pcap      # spojiť pcap súbory
```

### Exportovanie polí (CSV / JSON)

```bash
# Základný export polí do CSV
tshark -r file.pcap -T fields \
  -e frame.number \
  -e frame.time \
  -e ip.src \
  -e ip.dst \
  -e tcp.srcport \
  -e tcp.dstport \
  -e frame.len \
  -E header=y -E separator=, > output.csv

# JSON export
tshark -r file.pcap -T json > output.json

# PDML (XML)
tshark -r file.pcap -T pdml > output.xml
```

---

## Dekryptovanie

### HTTPS / TLS s SSL key logom

```bash
# 1. Nastaviť premennú prostredia pred spustením prehliadača:
export SSLKEYLOGFILE=~/ssl_keys.log
google-chrome &

# 2. V Wireshark:
# Edit → Preferences → Protocols → TLS
# → (Pre-)Master-Secret log filename: ~/ssl_keys.log

# CLI:
tshark -r file.pcap -o "tls.keylog_file:ssl_keys.log"
```

### WPA/WPA2 WiFi

```
# GUI: Edit → Preferences → Protocols → IEEE 802.11
#   → Decryption keys → Add
#   Typ: wpa-pwd
#   Kľúč: heslo:SSID

# Formáty kľúčov:
wpa-pwd   password:NetworkName
wpa-psk   a1b2c3d4...  (64 hex znakov – PSK)

# Podmienka: pcap musí obsahovať 4-way handshake!
# Filter pre handshake:
eapol
```

### RSA privátny kľúč (starší TLS bez PFS)

```
# Edit → Preferences → Protocols → TLS → RSA keys list → Add
#   IP:   server IP
#   Port: 443
#   Protocol: http
#   Key file: server.key

# Funguje IBA ak session nepoužíva DHE/ECDHE (forward secrecy)
```

---

## Štatistiky & Analýza

```
Statistics → Summary               – prehľad súboru, časy, počty paketov
Statistics → Protocol Hierarchy    – percentuálne zastúpenie protokolov
Statistics → Conversations         – zoznam všetkých spojení (TCP/UDP/IP/Ethernet)
Statistics → Endpoints             – zoznam všetkých IP/MAC adries
Statistics → IO Graphs             – grafický priebeh prevádzky v čase
Statistics → Flow Graph            – vizuálny diagram komunikácie medzi hostmi
Statistics → HTTP → Requests       – všetky HTTP požiadavky
Statistics → HTTP → Load Dist.     – rozdelenie záťaže medzi servermi
Statistics → DNS                   – prehľad DNS dopytov a odpovedí
Analyze → Expert Information       – automatické upozornenia na anomálie
```

---

## Colorizing Rules

```
# View → Colorize Packet List → Manage Color Rules

Čierna/červená   – TCP chyby, RST
Tmavočervená     – Bad TCP
Svetložltá       – SMB
Svetlomodrá      – DNS
Tmavozelená      – HTTP
Fialová          – TCP
Sivá             – TCP ACK

# Vlastné pravidlo:
# View → Colorize Packet List → New
#   Názov:   Suspicious DNS
#   Filter:  dns.qry.name matches "[a-f0-9]{30,}"
#   Farba:   červená pozadie
```

---

## tshark – príkazový riadok

### Základy

```bash
tshark -D                        # zoznam dostupných rozhraní
tshark -i eth0                   # zachytávať na eth0
tshark -i any                    # všetky rozhrania
tshark -i eth0 -c 100            # zachytiť 100 paketov
tshark -i eth0 -a duration:60    # zachytávať 60 sekúnd
tshark -i eth0 -f "port 80"      # capture filter
tshark -r file.pcap              # čítať súbor
tshark -r file.pcap -Y "http"    # display filter
tshark -r file.pcap -V           # verbose (všetky detaily)
tshark -r file.pcap -q           # quiet (len summary)
tshark -r file.pcap -z io,stat,0 # IO štatistiky
```

### One-liners

```bash
# Všetky unikátne IP adresy
tshark -r file.pcap -T fields -e ip.src | sort -u

# Top komunikujúce páry
tshark -r file.pcap -q -z conv,tcp

# Všetky HTTP requesty
tshark -r file.pcap -Y "http.request" -T fields \
  -e ip.src -e http.host -e http.request.method -e http.request.uri

# DNS requesty
tshark -r file.pcap -Y "dns.flags.response == 0" -T fields \
  -e ip.src -e dns.qry.name -e dns.qry.type

# Hľadanie reťazca v paketoch
tshark -r file.pcap -Y 'frame contains "password"'

# FTP prihlasovacie údaje
tshark -r file.pcap \
  -Y 'ftp.request.command == "USER" or ftp.request.command == "PASS"' \
  -T fields -e ip.src -e ftp.request.arg

# HTTP POST dáta
tshark -r file.pcap -Y "http.request.method == POST" -T fields \
  -e ip.src -e http.host -e http.request.uri -e http.file_data

# Počet paketov podľa protokolu
tshark -r file.pcap -q -z io,phs,

# Dlhé DNS mená (DNS tunel)
tshark -r file.pcap -Y "dns" -T fields -e dns.qry.name | awk 'length > 50'

# Extrahovanie všetkých URL
tshark -r file.pcap -Y "http.request" -T fields \
  -e http.host -e http.request.uri | sed 's/\t//' | sort -u

# Štatistiky portov
tshark -r file.pcap -q -z conv,tcp | sort -k9 -rn | head -20
```

---

## CTF – Typické scenáre

### Hľadanie credentialov

```
# HTTP Basic Auth (base64 encoded)
http.authorization contains "Basic"
# → Packet Details → Credentials → dekódovať base64

# FTP
ftp.request.command == "USER" or ftp.request.command == "PASS"

# Telnet (každý znak zvlášť)
telnet
# → Follow TCP Stream → ASCII → čítať

# SMTP autentifikácia
smtp.req.command == "AUTH"

# POP3
pop.request.command == "USER" or pop.request.command == "PASS"

# IMAP
imap and imap.request contains "LOGIN"

# HTTP POST formuláre
http.request.method == "POST"
# → Packet Details → HTML Form URL Encoded → Form item
```

### Hľadanie flagov

```bash
# Display filtre
frame contains "flag{"
frame contains "CTF{"
frame contains "FLAG{"

# Regex
frame matches "flag\\{[^}]+\\}"

# V HTTP tele
http contains "flag{"
http.file_data contains "flag"

# tshark
tshark -r file.pcap -Y 'frame contains "flag{"' \
  -T fields -e frame.number -e ip.src

# V exportovaných súboroch
# File → Export Objects → HTTP
grep -r "flag{" ./exported/
```

### Extrakcia súborov

```bash
# 1. File → Export Objects → HTTP

# 2. Filtrovať konkrétne typy
http.request.uri contains ".jpg"
http.content_type contains "image"
http.content_type contains "application/zip"
http.content_type contains "octet-stream"

# 3. tshark export
tshark -r file.pcap --export-objects http,./files/
tshark -r file.pcap --export-objects smb,./files/

# 4. Rekonštrukcia FTP súborov
# Follow TCP Stream na ftp-data → Save as Raw
ftp-data

# 5. Extrahovať raw bajty paketu
tshark -r file.pcap -Y "frame.number == 42" -T fields -e data.data
```

### Analýza sieťového skenovania

```
# Nmap ping sweep
icmp.type == 8

# SYN scan (half-open)
tcp.flags == 0x002

# Otvorené porty (SYN-ACK odpovede)
tcp.flags == 0x012

# Uzavreté porty (RST odpovede)
tcp.flags.rst == 1 and tcp.flags.ack == 1

# UDP scan – port unreachable = zatvorený port
icmp.type == 3 and icmp.code == 3

# Xmas scan (FIN+PSH+URG)
tcp.flags == 0x029

# NULL scan
tcp.flags == 0x000

# OS detection – TTL analýza
ip.ttl < 10
```

### Detekcia malvéru & C2

```
# Beacon (pravidelné spojenia)
# Statistics → IO Graphs → filter tcp.flags.syn == 1

# Nezvyčajné porty
tcp.dstport == 4444                  # Metasploit default
tcp.dstport == 1337
tcp.dstport == 31337

# DNS exfiltrácia
dns.qry.name matches "^[a-f0-9]{20,}"
dns.qry.name matches "\.onion\."

# ICMP tunel
icmp and data.len > 64

# HTTP beaconing
http.request.uri == "/check"
http.user_agent == ""                # prázdny User-Agent

# Base64 v URL (exfiltrácia)
http.request.uri matches "[A-Za-z0-9+/]{30,}={0,2}"

# PowerShell download
http.user_agent contains "PowerShell"
http.user_agent contains "WindowsPowerShell"
```

### WiFi analýza

```
wlan                             # všetka 802.11 prevádzka
wlan.fc.type_subtype == 0x00     # Association Request
wlan.fc.type_subtype == 0x08     # Beacon
wlan.fc.type_subtype == 0x0b     # Authentication
wlan.fc.type_subtype == 0x0c     # Deauthentication
eapol                            # WPA handshake (4-way)

# Filtrovanie konkrétnej SSID
wlan_mgt.ssid == "NetworkName"
wlan.bssid == aa:bb:cc:dd:ee:ff

# Deauth útok
wlan.fc.type_subtype == 0x0c and wlan.da == ff:ff:ff:ff:ff:ff
```

---

## Forenzná analýza – Postup

```
1. ORIENTÁCIA
   Statistics → Summary            – časy, veľkosť, rozhranie
   Statistics → Protocol Hierarchy – čo v súbore dominuje
   Statistics → Conversations      – kto s kým komunikuje
   Statistics → Endpoints          – aktívne hostitele

2. IDENTIFIKÁCIA ANOMÁLIÍ
   Analyze → Expert Information    – automatické varovania
   Hľadať: retransmisie, RST, chyby

3. FILTROVANIE PODOZRIVEJ PREVÁDZKY
   Pozrieť HTTP, DNS, ICMP, neštandardné porty

4. SLEDOVANIE STREAMOV
   Follow TCP/HTTP Stream na zaujímavé spojenia

5. EXTRAKCIA SÚBOROV
   File → Export Objects

6. HĽADANIE CITLIVÝCH DÁT
   Credentials, flagy, kľúče, tokeny

7. ČASOVÁ OS
   View → Time Display Format → Date and Time
   Zoradiť podľa času, hľadať súvislosti
```

---

## Klávesové skratky

| Skratka | Akcia |
|---|---|
| `Ctrl+E` | Spustiť/zastaviť zachytávanie |
| `Ctrl+K` | Nastavenia zachytávania |
| `Ctrl+O` | Otvoriť súbor |
| `Ctrl+S` | Uložiť |
| `Ctrl+Shift+S` | Uložiť ako |
| `Ctrl+F` | Hľadať v paketoch |
| `Ctrl+G` | Prejsť na paket číslo |
| `Ctrl+R` | Reload súbor |
| `Ctrl+D` | Zobraziť/skryť Packet Details |
| `Ctrl+B` | Zobraziť/skryť Packet Bytes |
| `Ctrl+1` | Zobraziť iba Packet List |
| `Ctrl+2` | Packet List + Details |
| `Ctrl+3` | Všetky tri panely |
| `Space` | Ďalší paket |
| `Backspace` | Predchádzajúci paket |
| `Ctrl+↓` | Ďalší zaujímavý paket |
| `Tab` | Nasledujúce pole v Packet Details |
| `Ctrl+M` | Označiť/odznačiť paket |
| `Ctrl+Shift+N` | Ďalší označený paket |
| `Ctrl+Alt+F` | Použiť ako filter |
| `=` | Zoom in |
| `-` | Zoom out |

---

## Dôležité polia pre filtrovanie

```
frame.number                     # číslo paketu
frame.time                       # čas
frame.len                        # celková dĺžka
frame.cap_len                    # zachytená dĺžka
frame.marked                     # označený paket

eth.src / eth.dst                # MAC adresy
eth.type                         # 0x0800=IPv4, 0x0806=ARP

ip.src / ip.dst
ip.proto                         # 6=TCP, 17=UDP, 1=ICMP
ip.ttl
ip.len
ip.flags.df                      # Don't Fragment bit
ip.fragment                      # fragmenty

tcp.srcport / tcp.dstport
tcp.seq / tcp.ack                # sekvenčné čísla
tcp.window_size
tcp.payload
tcp.stream                       # ID TCP streamu

udp.srcport / udp.dstport
udp.length

data                             # surové dáta (payload)
data.data                        # hex payload
data.len                         # dĺžka payloadu
```

---

## Referencia – DNS typy záznamov

| Typ | Číslo | Popis |
|---|---|---|
| A | 1 | IPv4 adresa |
| NS | 2 | Name server |
| MX | 15 | Mail exchange |
| TXT | 16 | Textový záznam |
| AAAA | 28 | IPv6 adresa |
| SRV | 33 | Service record |
| ANY | 255 | Všetky záznamy |

---

## Referencia – TCP flagy

| Flag | Bit | Popis |
|---|---|---|
| FIN | 0x01 | Koniec spojenia |
| SYN | 0x02 | Synchronizácia |
| RST | 0x04 | Reset spojenia |
| PSH | 0x08 | Push (odoslať ihneď) |
| ACK | 0x10 | Potvrdenie |
| URG | 0x20 | Urgentné dáta |
| ECE | 0x40 | ECN Echo |
| CWR | 0x80 | Congestion Window Reduced |

---

## Nástroje dopĺňajúce Wireshark

```bash
# Konverzia formátov
editcap -F pcap input.pcapng output.pcap     # pcapng → pcap
editcap -i 60 big.pcap split.pcap            # rozdeliť po 60 sek.
mergecap -w merged.pcap a.pcap b.pcap        # spojiť súbory

# Rýchla analýza
capinfos file.pcap                           # info o súbore
tcpdump -r file.pcap -nn -v                  # čítať bez DNS lookupov
tcpdump -r file.pcap -nn "port 80"           # s filtrom
tcpdump -i eth0 -w out.pcap                  # zachytávať

# Zeek (Bro) – generovanie logov zo pcap
zeek -r file.pcap

# Scapy – Python knižnica na analýzu/tvorbu paketov
from scapy.all import *
pkts = rdpcap("file.pcap")
pkts[0].show()
```
