---
layout: notes
title: "Defensive Playbook – SOC"
folder: "Playbooks"
tags: [playbook, defensive, soc, blue-team, incident-response, log-analysis, threat-hunting]
---

# 🛡️ Defensive Playbook – SOC

---

## 📋 Obsah

1. [Triáž alertu](#triáž-alertu)
2. [Analýza sieťovej prevádzky](#analýza-sieťovej-prevádzky)
3. [Analýza logov](#analýza-logov)
4. [Analýza podozrivého súboru / procesu](#analýza-podozrivého-súboru--procesu)
5. [Incident Response](#incident-response)
6. [Threat Hunting](#threat-hunting)
7. [Windows – Dôležité Event IDs](#windows--dôležité-event-ids)
8. [Linux – Dôležité logy](#linux--dôležité-logy)
9. [Indikátory kompromitácie (IoC)](#indikátory-kompromitácie-ioc)
10. [Nástroje](#nástroje)

---

## Triáž alertu

> Cieľ: Rýchlo rozhodnúť – True Positive alebo False Positive? Aký je rozsah?

```
Alert príde → Triáž → Vyšetrovanie → Response → Dokumentácia → Lessons Learned
```

### Checklist – Prvých 5 minút

- [ ] Prečítaj alert – čo presne spustilo pravidlo?
- [ ] True Positive alebo False Positive?
- [ ] Aký je severity level? (Critical / High / Medium / Low)
- [ ] Ktoré systémy / používatelia sú ovplyvnení?
- [ ] Je útok stále aktívny?
- [ ] Eskalovať alebo riešiť sám?
- [ ] Otvor ticket / zaznač začiatok vyšetrovania

### Severity Assessment

| Severity | Príklady | Reakcia |
|----------|----------|---------|
| Critical | Ransomware, data exfil, DC kompromit. | Okamžitá izolácia, eskalácia |
| High | Malware detekcia, brute force úspech, lateral movement | Vyšetrovanie < 1 hod |
| Medium | Port scan, failed login spike, podoz. process | Vyšetrovanie < 4 hod |
| Low | Policy violation, recon aktivity | Vyšetrovanie < 24 hod |

---

## Analýza sieťovej prevádzky

### Identifikácia podozrivej komunikácie

```bash
# Zachytávanie trafficu
tcpdump -i eth0 -w capture.pcap
tcpdump -i eth0 host <IP> -w capture.pcap
tcpdump -i eth0 port 443 -w capture.pcap

# Rýchla analýza
tcpdump -r capture.pcap
tcpdump -r capture.pcap -nn              # bez DNS rezolúcie
tcpdump -r capture.pcap 'host <IP>'

# Netstat – aktívne spojenia
netstat -tulnp                           # Linux
netstat -ano                             # Windows
ss -tulnp                                # Linux alternatíva

# Tshark (command-line Wireshark)
tshark -r capture.pcap -Y "http"
tshark -r capture.pcap -Y "dns"
tshark -r capture.pcap -T fields -e ip.src -e ip.dst -e tcp.dstport | sort | uniq -c | sort -rn
```

### Čo hľadať v trafficu

| Vzorec | Možná hrozba |
|--------|-------------|
| Pravidelné intervaly (beacon) | C2 komunikácia |
| Veľký objem dát von | Exfiltrácia |
| Komunikácia na neobvyklé porty | Tunel / backdoor |
| DNS na externé servery | DNS tunneling, C2 |
| Nešifrovaný HTTP s dátami | Exfiltrácia, malware |
| SMB na externé IP | Lateral movement / worm |
| ICMP s dátami | ICMP tunneling |

### Beaconing detekcia

```bash
# Pozri na pravidelnosť spojení (zeek / suricata logy)
cat conn.log | awk '{print $3, $5, $7}' | sort

# Tshark – export IP komunikácie
tshark -r capture.pcap -T fields -e frame.time_relative -e ip.dst -e ip.len \
  | grep "<suspicious-IP>"
```

### Threat Intelligence – overenie IP / domény

```
VirusTotal:    https://virustotal.com
AbuseIPDB:     https://abuseipdb.com
Shodan:        https://shodan.io
URLScan:       https://urlscan.io
OTX:           https://otx.alienvault.com
IBM X-Force:   https://exchange.xforce.ibmcloud.com
ThreatFox:     https://threatfox.abuse.ch
```

```bash
# Rýchle overenie cez curl (AbuseIPDB API)
curl -G https://api.abuseipdb.com/api/v2/check \
  --data-urlencode "ipAddress=<IP>" \
  -H "Key: YOUR_API_KEY" \
  -H "Accept: application/json"
```

---

## Analýza logov

### Umiestnenie logov

| Systém | Umiestnenie |
|--------|-------------|
| Linux auth | `/var/log/auth.log` (Debian) / `/var/log/secure` (RHEL) |
| Linux syslog | `/var/log/syslog` / `/var/log/messages` |
| Linux audit | `/var/log/audit/audit.log` |
| Apache | `/var/log/apache2/access.log`, `error.log` |
| Nginx | `/var/log/nginx/access.log`, `error.log` |
| Windows Security | `Event Viewer → Windows Logs → Security` |
| Windows System | `Event Viewer → Windows Logs → System` |
| Windows PowerShell | `Event Viewer → Apps → PowerShell` |
| Windows Sysmon | `Event Viewer → Apps → Sysmon` |

### Linux – Analýza logov

```bash
# Neúspešné prihlasovania
grep "Failed password" /var/log/auth.log
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Úspešné prihlasovania
grep "Accepted password\|Accepted publickey" /var/log/auth.log

# SSH brutforce detekcia – IP s najviac pokusmi
grep "Failed password" /var/log/auth.log | grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn | head -20

# Sudo príkazy
grep "sudo" /var/log/auth.log
grep "COMMAND" /var/log/auth.log

# Nové používateľské účty
grep "useradd\|adduser\|usermod" /var/log/auth.log

# Zmeny súborov (ak je auditd)
ausearch -f /etc/passwd
ausearch -f /etc/sudoers
ausearch -k privileged

# Prihlasovania za posledný deň
last -F | head -30
lastb | head -20                         # neúspešné

# Bežiace procesy a sieťové spojenia
ps aux --sort=-%cpu | head -20
lsof -i -n -P
```

### Windows – Analýza logov (PowerShell)

```powershell
# Posledné neúspešné prihlasovania (4625)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 50 |
  Select TimeCreated, Message | Format-List

# Posledné úspešné prihlasovania (4624)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 50 |
  Select TimeCreated, Message | Format-List

# Nové procesy (4688)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688} -MaxEvents 100 |
  Select TimeCreated, Message | Format-List

# Nové scheduled tasks (4698)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4698} |
  Select TimeCreated, Message | Format-List

# Nové služby (7045)
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7045} |
  Select TimeCreated, Message | Format-List

# PowerShell history – podozrivé príkazy
Get-WinEvent -LogName 'Microsoft-Windows-PowerShell/Operational' |
  Where-Object Message -match "Invoke-|DownloadString|IEX|EncodedCommand|base64" |
  Select TimeCreated, Message | Format-List

# Filtruj logy podľa času
Get-WinEvent -FilterHashtable @{LogName='Security'; StartTime='2024-01-01 00:00:00'; EndTime='2024-01-02 00:00:00'}
```

### Web Server – Analýza access logov

```bash
# Top IP adresy
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -20

# Top URL požiadavky
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -20

# HTTP 4xx / 5xx chyby
grep ' 4[0-9][0-9] \| 5[0-9][0-9] ' access.log | awk '{print $1, $7, $9}' | sort | uniq -c | sort -rn

# Podozrivé User-Agenty
grep -i "sqlmap\|nikto\|nmap\|masscan\|dirbuster\|gobuster\|hydra\|metasploit\|python-requests\|curl/\|wget/" access.log

# SQL Injection pokusy
grep -i "select\|union\|insert\|drop\|--\|1=1\|' OR\|xp_cmdshell" access.log

# LFI / Path traversal pokusy
grep -i "\.\./\|\.\.%2f\|%2e%2e\|etc/passwd\|etc/shadow\|/proc/self" access.log

# Webshell pokusy
grep -i "cmd=\|exec=\|command=\|shell=\|passthru\|system(\|eval(" access.log

# Skenery / brute force – veľa 404
awk '$9 == 404 {print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

---

## Analýza podozrivého súboru / procesu

### Statická analýza súboru

```bash
# Základné info
file suspicious_file
xxd suspicious_file | head -20           # hex dump
strings suspicious_file | head -50       # čitateľné reťazce
strings suspicious_file | grep -i "http\|url\|ftp\|cmd\|powershell\|base64"

# Hash súboru
md5sum suspicious_file
sha256sum suspicious_file
# → Over na VirusTotal

# PE hlavička (Windows executable)
objdump -f suspicious_file
exiftool suspicious_file

# Packed / obfuskovaný binary?
strings suspicious_file | wc -l          # málo stringov = packed
upx -t suspicious_file                   # UPX packer check
```

### Online sandboxing

```
Any.run:        https://app.any.run
Hybrid Analysis: https://hybrid-analysis.com
Joe Sandbox:    https://joesandbox.com
Triage:         https://tria.ge
VirusTotal:     https://virustotal.com/gui/file (upload)
```

### Analýza procesov (Windows)

```powershell
# Bežiace procesy
Get-Process | Sort-Object CPU -Descending | Select -First 20
Get-Process | Select Name, Id, Path, Company | Format-Table

# Podozrivé procesy – bez cesty
Get-Process | Where-Object {$_.Path -eq $null} | Select Name, Id

# Procesy s network spojeniami
Get-NetTCPConnection -State Established |
  Select LocalAddress, LocalPort, RemoteAddress, RemotePort, OwningProcess |
  ForEach-Object {
    $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
    [PSCustomObject]@{
      LocalPort = $_.LocalPort
      RemoteAddress = $_.RemoteAddress
      RemotePort = $_.RemotePort
      ProcessName = $proc.Name
      PID = $_.OwningProcess
    }
  }

# Autostart položky
Get-CimInstance Win32_StartupCommand | Select Name, Command, Location
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run

# Scheduled tasks
Get-ScheduledTask | Where-Object State -ne 'Disabled' | Select TaskName, TaskPath, State
```

### Analýza procesov (Linux)

```bash
# Bežiace procesy
ps aux --sort=-%cpu
ps aux --sort=-%mem

# Procesy so sieťovými spojeniami
lsof -i -n -P
ss -tulnp

# Podozrivé – procesy z /tmp alebo /dev/shm
ls -la /tmp /dev/shm /var/tmp
lsof | grep "tmp\|/dev/shm"

# Otvorené súbory procesu
lsof -p <PID>

# Mapa pamäte procesu
cat /proc/<PID>/maps
cat /proc/<PID>/cmdline | tr '\0' ' '

# Cron joby
crontab -l
cat /etc/crontab
ls /etc/cron*
cat /var/spool/cron/crontabs/*

# Persistence – startup
ls /etc/init.d/
ls /etc/systemd/system/
cat /etc/rc.local
```

---

## Incident Response

### IR Fázy

```
1. PRÍPRAVA       → Plány, nástroje, školenia pred incidentom
2. IDENTIFIKÁCIA  → Čo sa stalo? Kedy? Aký rozsah?
3. IZOLÁCIA       → Zastav šírenie – odpoj od siete
4. ERADIKÁCIA     → Odstráň malware, backdoory, kompromitované účty
5. OBNOVA         → Obnov zo zálohy, patch, monitoruj
6. DOKUMENTÁCIA   → Zaznač všetko → report → lessons learned
```

### Izolácia kompromitovaného systému

```bash
# Linux – odpoj od siete (zachovaj dôkazy!)
ip link set eth0 down
iptables -I INPUT -j DROP
iptables -I OUTPUT -j DROP

# Záloha dôkazov pred izoláciou
# Zachyť RAM (ak možné)
avml /tmp/memory.lime
insmod lime.ko "path=/tmp/memory.lime format=lime"

# Snapshot disk
dd if=/dev/sda of=/mnt/evidence/disk.img bs=4M
sha256sum /dev/sda > /mnt/evidence/disk.sha256

# Linux – aktívne artefakty
ps aux > /tmp/ir/processes.txt
netstat -tulnp > /tmp/ir/connections.txt
last -F > /tmp/ir/logins.txt
find / -mtime -1 -type f 2>/dev/null > /tmp/ir/recent_files.txt
```

```powershell
# Windows – odpoj od siete
Disable-NetAdapter -Name "Ethernet" -Confirm:$false

# Zachyť artefakty
Get-Process > C:\IR\processes.txt
Get-NetTCPConnection > C:\IR\connections.txt
Get-WinEvent -LogName Security -MaxEvents 1000 | Export-Csv C:\IR\security_events.csv

# Scheduled tasks
Get-ScheduledTask | Export-Csv C:\IR\scheduled_tasks.csv

# Autorun
reg export HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run C:\IR\run_hklm.reg
reg export HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run C:\IR\run_hkcu.reg
```

### Indikátory Lateral Movement

```bash
# Windows – hľadaj Event ID 4648 (explicit credentials) a 4624 Type 3 (network logon)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4648} | Select TimeCreated, Message

# Linux – SSH prihlasovania z interných IP
grep "Accepted" /var/log/auth.log | grep -v "known-good-ip"

# SMB aktivity
grep "smb\|samba" /var/log/syslog

# Pass-the-Hash / Pass-the-Ticket indikátory
# Event 4624 s LogonType=3 a NtlmV1
# Kerberoas TGS requesty na nezvyčajné služby (4769)
```

### Ransomware Response

```
OKAMŽITE:
1. Izoluj všetky infikované systémy od siete
2. Izoluj NAS / file servery
3. Skontroluj zálohy – sú nedotknuté?
4. Identifikuj Patient Zero – prvý infikovaný systém
5. Identifikuj ransomware variant (ID Ransomware: https://id-ransomware.malwarehunterteam.com)
6. NEPLAŤIŤ výkupné bez konzultácie s managementom / právnikmi
7. Nahlásiť príslušným orgánom (NÚKIB, polícia)
```

```bash
# Identifikuj zašifrované súbory
find / -name "*.locked" -o -name "*.encrypted" -o -name "*.crypted" 2>/dev/null
find / -name "HOW_TO_DECRYPT*" -o -name "README_DECRYPT*" -o -name "*RANSOM*" 2>/dev/null

# Sleduj file system aktivitu (Linux)
inotifywait -m -r /home /var /etc --format '%T %w %f %e' --timefmt '%Y-%m-%d %H:%M:%S'

# Hľadaj shadow copy mazanie (Windows)
Get-WinEvent -FilterHashtable @{LogName='System'} |
  Where-Object Message -match "vssadmin\|shadowcopy\|wbadmin" | Select TimeCreated, Message
```

---

## Threat Hunting

> Proaktívne hľadanie hrozieb ktoré preskočili detekciu.

### Hypotézy a postup

```
1. Definuj hypotézu (napr. "Útočník používa LOLBins pre persistence")
2. Urči dáta ktoré potrebuješ (process logs, network logs...)
3. Hľadaj anomálie / IOC
4. Validuj nálezy
5. Zdokumentuj a vytvor detection rule
```

### LOLBins – Living off the Land

```powershell
# Podozrivé použitie legitímnych nástrojov
# Hľadaj v process creation logoch (Sysmon EID 1 alebo Security EID 4688)

# certutil – download súborov
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688} |
  Where-Object Message -match "certutil.*urlcache\|certutil.*decode"

# bitsadmin – download
Where-Object Message -match "bitsadmin.*transfer\|bitsadmin.*download"

# mshta – execute HTA
Where-Object Message -match "mshta.*http\|mshta.*vbscript\|mshta.*javascript"

# regsvr32 – squiblydoo
Where-Object Message -match "regsvr32.*/s.*scrobj"

# rundll32 – execute DLL
Where-Object Message -match "rundll32.*javascript\|rundll32.*vbscript"

# wscript / cscript – script execution
Where-Object Message -match "wscript.*\.js\|cscript.*\.vbs"

# PowerShell obfuskácia
Where-Object Message -match "EncodedCommand\|-enc\|IEX\|Invoke-Expression\|DownloadString\|Net.WebClient"
```

### Persistence Hunting

```powershell
# Nové scheduled tasks (porovnaj s baseline)
Get-ScheduledTask | Where-Object {$_.Date -gt (Get-Date).AddDays(-7)} | Select TaskName, Date, Actions

# Nové služby
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7045} -MaxEvents 50 | Select TimeCreated, Message

# Registry run keys – porovnaj s baseline
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce

# WMI persistence
Get-WMIObject -Namespace root\subscription -Class __EventFilter
Get-WMIObject -Namespace root\subscription -Class __EventConsumer
Get-WMIObject -Namespace root\subscription -Class __FilterToConsumerBinding
```

```bash
# Linux persistence hunting
# Nové cron joby (posledných 7 dní)
find /etc/cron* /var/spool/cron -newer /etc/passwd -ls 2>/dev/null

# Nové systemd services
find /etc/systemd/system -newer /etc/passwd -name "*.service" 2>/dev/null

# Modifikované SSH authorized_keys
find /home /root -name "authorized_keys" -newer /etc/passwd 2>/dev/null

# SUID bity pridané nedávno
find / -perm -4000 -newer /etc/passwd -type f 2>/dev/null

# Nové binary v systémových adresároch
find /bin /usr/bin /sbin /usr/sbin -newer /etc/passwd -type f 2>/dev/null
```

### Network Anomaly Hunting

```bash
# DNS – neobvyklé domény (dlhé názvy = možný DGA alebo DNS tunneling)
cat dns.log | awk '{print $9}' | awk -F'.' '{print length($0), $0}' | sort -rn | head -20

# Beaconing – pravidelné spojenia na rovnakú IP
cat conn.log | awk '{print $3, $5}' | sort | uniq -c | sort -rn | head -20

# Neobvyklé porty pre komunikáciu
cat conn.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -30

# Veľké prenosy von (možná exfiltrácia)
cat conn.log | awk '{if ($10+0 > 10000000) print $3, $5, $10}' | sort -k3 -rn
```

---

## Windows – Dôležité Event IDs

### Prihlasovanie a autentifikácia

| ID | Udalosť | Čo hľadať |
|----|---------|-----------|
| 4624 | Úspešné prihlásenie | LogonType 3/10 z neznámych IP |
| 4625 | Neúspešné prihlásenie | Spike = brute force |
| 4648 | Prihlásenie s explicit credentials | RunAs, lateral movement |
| 4672 | Special privileges (admin) | Kto a kedy dostal admin |
| 4768 | Kerberos TGT request | Neúspešné = bad password |
| 4769 | Kerberos TGS request | Kerberoasting ak veľa naraz |
| 4771 | Kerberos pre-auth failed | Password spray |
| 4776 | NTLM auth | Pass-the-Hash indikátor |

### Správa účtov

| ID | Udalosť |
|----|---------|
| 4720 | Nový používateľský účet |
| 4722 | Účet aktivovaný |
| 4723 | Zmena hesla (používateľ) |
| 4724 | Reset hesla (admin) |
| 4728 | Pridaný do security group |
| 4732 | Pridaný do lokálnej skupiny |
| 4756 | Pridaný do universal group |

### Procesy a systém

| ID | Udalosť |
|----|---------|
| 4688 | Nový proces spustený |
| 4689 | Proces ukončený |
| 4698 | Scheduled task vytvorený |
| 4702 | Scheduled task upravený |
| 4704 | Privilégium pridané účtu |
| 7045 | Nová služba nainštalovaná |
| 7036 | Služba zmenila stav |

### Sysmon Event IDs

| ID | Udalosť |
|----|---------|
| 1 | Process creation (s command line) |
| 3 | Network connection |
| 7 | Image/DLL loaded |
| 8 | CreateRemoteThread (injekcia) |
| 10 | ProcessAccess (LSASS dump!) |
| 11 | File created |
| 12/13 | Registry event |
| 22 | DNS query |
| 25 | Process tampering |

### LogonType hodnoty

| Typ | Popis | Riziko |
|-----|-------|--------|
| 2 | Interaktívny (lokálny) | Normálny |
| 3 | Sieťový (SMB, net use) | Sledovať |
| 4 | Batch (scheduled task) | Sledovať |
| 5 | Služba | Normálny |
| 7 | Odomknutie | Normálny |
| 8 | NetworkCleartext (heslo plaintext!) | Vysoké riziko |
| 9 | NewCredentials (RunAs) | Sledovať |
| 10 | RemoteInteractive (RDP) | Sledovať |

---

## Linux – Dôležité logy

### Súbory a čo hľadať

| Log | Kľúčové vzorce |
|-----|----------------|
| `/var/log/auth.log` | `Failed password`, `Invalid user`, `Accepted`, `sudo` |
| `/var/log/syslog` | Systémové udalosti, cron, kernel |
| `/var/log/kern.log` | Kernel správy, moduly |
| `/var/log/audit/audit.log` | Syscall, file access, execve |
| `/var/log/apache2/access.log` | 4xx/5xx, skenery, injekcie |
| `/var/log/nginx/error.log` | Chyby, podozrivé requesty |
| `/var/log/mail.log` | Spam, relay, phishing |
| `~/.bash_history` | Príkazy používateľa |
| `/root/.bash_history` | Rootové príkazy |

### Užitočné one-linery

```bash
# Kto je prihlásený teraz
w
who
last -F | head -20

# Posledná aktivita používateľov
lastlog

# Neúspešné SSH pokusy za posledných 24h
grep "Failed password" /var/log/auth.log | grep "$(date '+%b %d')" | wc -l

# Top útočné IP dnes
grep "Failed password" /var/log/auth.log | grep "$(date '+%b %d')" | \
  grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn | head -10

# Súbory modifikované za posledných 24h
find /etc /home /var/www -mtime -1 -type f 2>/dev/null

# Hľadaj webshell
find /var/www -name "*.php" -newer /var/www/html/index.php 2>/dev/null
grep -rn "eval\|base64_decode\|system\|passthru\|shell_exec" /var/www/ --include="*.php"

# Neobvyklé SUID bity
find / -perm -4000 -type f 2>/dev/null | sort
```

---

## Indikátory kompromitácie (IoC)

### Typy IoC a kde hľadať

| IoC typ | Zdroje | Nástroje |
|---------|--------|---------|
| IP adresa | Firewall logy, netflow, proxy | VirusTotal, AbuseIPDB, Shodan |
| Doména / URL | DNS logy, proxy logy, web logy | VirusTotal, URLScan, URLVoid |
| File hash (MD5/SHA) | EDR, AV logy, SIEM | VirusTotal, Hybrid Analysis |
| Email adresa | Mail logy, phishing reports | VirusTotal, Have I Been Pwned |
| User-Agent | Web server logy, proxy | Grep v logoch |
| Registry key | Windows Event logy, EDR | Autoruns, SIEM |
| Mutex | Memory forensics, sandbox | Hybrid Analysis, Any.run |
| YARA rule | Súbory na disku / v pamäti | YARA, yarGen |

### IoC Management

```bash
# Rýchla kontrola IP v logoch
grep -r "suspicious.ip.here" /var/log/

# Grep pre viacero IoC naraz (zo súboru)
grep -Ff iocs.txt /var/log/apache2/access.log
grep -Ff iocs.txt /var/log/auth.log

# YARA scan súborov
yara rules.yar /path/to/scan -r

# Windows – hľadaj hash v procesoch
Get-Process | ForEach-Object {
  $hash = (Get-FileHash $_.Path -ErrorAction SilentlyContinue).Hash
  if ($hash -eq "MALICIOUS_HASH") { Write-Host "FOUND: $($_.Name) PID: $($_.Id)" }
}
```

---

## Nástroje

### SIEM / Log Analysis

```
Splunk        – enterprise SIEM, SPL jazyk
Elastic/ELK   – open source, KQL
Graylog       – open source log management
Wazuh         – open source SIEM + EDR
QRadar        – IBM enterprise SIEM
Sentinel      – Microsoft Azure SIEM
```

### Network Analysis

```
Wireshark     – GUI packet analyzer
Zeek (Bro)    – network security monitor, logy
Suricata      – IDS/IPS, network monitoring
Arkime        – full packet capture a search
NetworkMiner  – network forensics
```

### Endpoint / Forensics

```
Velociraptor  – endpoint visibility, DFIR
Volatility    – memory forensics
Autopsy       – disk forensics GUI
FTK Imager    – disk imaging
KAPE          – artifact collection
Sysinternals  – Process Explorer, Autoruns, TCPView
```

### Threat Intel

```
MISP          – threat intel sharing platform
OpenCTI       – cyber threat intelligence
MITRE ATT&CK  – https://attack.mitre.org
Sigma         – generic SIEM rules
YARA          – malware pattern matching
```

### Rýchla referencia – MITRE ATT&CK Taktiky

| ID | Taktika | Príklady techník |
|----|---------|-----------------|
| TA0001 | Initial Access | Phishing, exploit public app |
| TA0002 | Execution | PowerShell, cmd, WMI |
| TA0003 | Persistence | Scheduled tasks, registry run keys |
| TA0004 | Privilege Escalation | Token impersonation, SUID |
| TA0005 | Defense Evasion | Obfuskácia, LOLBins, log clearing |
| TA0006 | Credential Access | LSASS dump, Kerberoasting |
| TA0007 | Discovery | Network scan, account enum |
| TA0008 | Lateral Movement | Pass-the-Hash, RDP, SMB |
| TA0009 | Collection | Keylogging, screen capture |
| TA0010 | Exfiltration | DNS tunneling, HTTPS exfil |
| TA0011 | C2 | Beaconing, domain fronting |
| TA0040 | Impact | Ransomware, data destruction |
