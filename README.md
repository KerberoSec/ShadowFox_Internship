<div align="center">

# ShadowFox Cyber Security Internship

**Beginner Level Security Assessment**

Professional penetration testing assessment conducted as part of the ShadowFox Cyber Security Internship Program.

</div>

---

## Overview

This repository contains the complete **Beginner Level Security Assessment** performed as part of the **ShadowFox Cyber Security Internship Program**. The assessment was conducted on an intentionally vulnerable web application hosted at `testasp.vulnweb.com`, a legal practice target operated by Acunetix specifically for security training.

The objective was to simulate a real world **black box penetration testing scenario** and identify security weaknesses using industry standard tools and methodologies.

> **Assessment Scope**
>
> This assessment was performed only against `testasp.vulnweb.com`, an intentionally vulnerable practice application provided by Acunetix for legal security training. No production systems or unauthorized targets were accessed.

## Contents

1. [Target Information](#target-information)
2. [Methodology](#methodology)
3. [Tasks Summary](#tasks-summary)
4. [Task 1: Open Port Discovery Using Nmap](#task-1-open-port-discovery-using-nmap)
5. [Task 2: Directory Enumeration Using Gobuster and FFUF](#task-2-directory-enumeration-using-gobuster-and-ffuf)
6. [Task 3: Plaintext Credential Capture Using Wireshark](#task-3-plaintext-credential-capture-using-wireshark)
7. [Consolidated Vulnerability Assessment](#consolidated-vulnerability-assessment)
8. [Recommendations](#recommendations)
9. [Tools Used](#tools-used)
10. [Repository Structure](#repository-structure)
11. [Final Statistics](#final-statistics)
12. [Learning Outcomes](#learning-outcomes)
13. [Report](#report)
14. [Legal and Ethical Notice](#legal-and-ethical-notice)
15. [Contact](#contact)

---

## Target Information

| Parameter | Detail |
|:----------|:-------|
| **Target URL** | `http://testasp.vulnweb.com` |
| **Target IP** | `44.238.29.244` |
| **Reverse DNS** | `ec2-44-238-29-244.us-west-2.compute.amazonaws.com` |
| **Web Server** | Microsoft IIS 8.5 |
| **Operating System** | Windows Server |
| **Application Stack** | Classic ASP (Active Server Pages) |
| **Hosting** | Amazon Web Services EC2 (US West 2, Oregon) |
| **Protocol** | HTTP only. No HTTPS detected. |

---

## Methodology

The assessment followed the **NIST SP 800 115** four phase penetration testing lifecycle, aligned with the **OWASP Testing Guide v4**:

```
Phase 1: Planning and Reconnaissance
        |
Phase 2: Discovery and Enumeration
        |
Phase 3: Exploitation and Verification
        |
Phase 4: Reporting and Documentation
```

---

## Tasks Summary

| # | Task | Tools | Key Finding | Status |
|:--|:-----|:------|:------------|:-------|
| 1 | Open Port Discovery | Nmap 7.98 | Port 80 open with IIS 8.5 and missing security headers | Completed |
| 2 | Directory Enumeration | Gobuster 3.8.2 and FFUF 2.1 | 14 directories found including sensitive system folders | Completed |
| 3 | Credential Interception | Wireshark | Plaintext credentials captured from HTTP POST packets | Completed |

---

## Task 1: Open Port Discovery Using Nmap

### Objective
Identify all open TCP ports and running services on the target server.

### Command Used
```bash
nmap -sC -sV -Pn testasp.vulnweb.com
```

| Flag | Purpose |
|:-----|:--------|
| `-sC` | Run default NSE scripts (banner grabbing, header analysis, cookie checks) |
| `-sV` | Detect service versions |
| `-Pn` | Skip host discovery and treat target as online regardless of ICMP response |

### Nmap Output
```
Starting Nmap 7.98 at 2026-03-30 23:30 +0530
Nmap scan report for testasp.vulnweb.com (44.238.29.244)
Host is up (0.33s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 8.5
|_http-server-header: Microsoft-IIS/8.5
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: acuforum forums
| http-cookie-flags:
|   /:
|     ASPSESSIONIDCQDABSCC:
|_      httponly flag not set
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address (1 host up) scanned in 41.33 seconds
```

### Findings

| Finding | Severity | Security Implication |
|:--------|:---------|:---------------------|
| Port 80/tcp open (HTTP only) | High | All data transmitted in cleartext. Credentials, cookies, and personal information fully exposed. |
| Microsoft IIS 8.5 (EOL Oct 2023) | High | No vendor patches for newly discovered CVEs |
| HTTP TRACE method enabled | Medium | Enables Cross Site Tracing (XST) attacks to steal session tokens |
| HttpOnly flag missing on session cookie | Medium | Session cookies accessible to JavaScript and exploitable via XSS |

---

## Task 2: Directory Enumeration Using Gobuster and FFUF

### Objective
Discover hidden, unlisted, or unlinked directories on the target web server.

### Commands Used

```bash
# Gobuster: Primary enumeration
gobuster dir -u http://testasp.vulnweb.com/ -w /usr/share/wordlists/dirb/common.txt -t 50

# FFUF: Cross validation
ffuf -u http://testasp.vulnweb.com/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

### 14 Directories Discovered

| Directory | HTTP Status | Risk | Security Concern |
|:----------|:------------|:-----|:-----------------|
| `/_vti_cnf/` | 301 | Medium | FrontPage server extensions configuration may expose internal path information |
| `/aspnet_client/` | 301 | Low | ASP.NET client side scripts; may contain outdated JavaScript libraries |
| `/avatars/` | 301 | Medium | User uploaded files; potential path traversal vector |
| `/cgi-bin/` | 301 then 403 | High | CGI directory confirmed and historically exploitable for RCE |
| `/html/` | 301 | Low | Static content; may contain unintended public files |
| `/images/` | 301 | Low | Image assets directory |
| `/jscripts/` | 301 | Medium | JavaScript files may contain hardcoded keys or internal endpoint URLs |
| `/robots.txt` | 200 | Low | Reveals sensitive paths meant to be hidden from search engines |
| `/templates/` | 301 | Medium | Template files expose application structure; potential SSTI target |
| `/t/` and `/T/` | 301 | Medium | Ambiguous paths warrant further investigation |

> **Note:** Both `/images/` and `/Images/` returning identical responses confirms a **Windows NTFS case insensitive file system**, consistent with the IIS 8.5 identification from Task 1.

---

## Task 3: Plaintext Credential Capture Using Wireshark

### Objective
Intercept live network traffic during an active login session and extract credentials transmitted over unencrypted HTTP.

### Tool
**Wireshark** configured to capture all packets on the `eth0` interface.

### Filter Used
```
http.request.method == "POST"
```

### Procedure

1. Launch Wireshark with superuser privileges: `sudo wireshark`
2. Select the `eth0` interface and start the capture
3. Open Firefox and navigate to `http://testasp.vulnweb.com/`
4. Register a new account at `/register.asp`
5. Log in at `/Login.asp`
6. Apply the POST filter in Wireshark
7. Inspect captured packet payloads

### Captured Registration Packet
```http
POST /register.asp HTTP/1.1
Host: testasp.vulnweb.com
Content-Type: application/x-www-form-urlencoded
Cookie: ASPSESSIONIDCQDABSCC=GNPAOENAJCMDFIDFDCIDLFKO

tfUName=hacker1233&tfRName=hacker1233&tfEmail=hacker%401233&tfUPass=hacker%401233
```

### Captured Login Packet
```http
POST /Login.asp?RetURL= HTTP/1.1
Host: testasp.vulnweb.com
Content-Type: application/x-www-form-urlencoded
Cookie: ASPSESSIONIDCQDABSCC=GNPAOENAJCMDFIDFDCIDLFKO

tfUName=hacker123&tfUPass=hacker1233%40gmail.com
```

### Decoded Credentials

| Request | Field | Captured Value | Decoded |
|:--------|:------|:---------------|:--------|
| Registration | Username | `hacker1233` | hacker1233 |
| Registration | Password | `hacker%401233` | hacker@1233 |
| Login | Username | `hacker123` | hacker123 |
| Login | Password | `hacker1233%40gmail.com` | hacker1233@gmail.com |

> **Important:** URL encoding is not encryption. The `%40` substitution for `@` is trivially reversible and does not provide security. The session cookie `ASPSESSIONIDCQDABSCC=GNPAOENAJCMDFIDFDCIDLFKO` was also visible in every captured request, enabling session hijacking.

---

## Consolidated Vulnerability Assessment

| ID | Vulnerability | Severity | CVSS | Discovered In |
|:---|:--------------|:---------|:-----|:--------------|
| V01 | Insecure HTTP Transmission (No HTTPS) | Critical | 9.1 | Task 1 and 3 |
| V02 | Plaintext Credential Submission | Critical | 9.4 | Task 3 |
| V03 | Session Cookie Transmitted in Cleartext | High | 8.1 | Task 3 |
| V04 | End of Life Web Server (IIS 8.5) | High | 7.5 | Task 1 |
| V05 | Missing HttpOnly Flag on Session Cookie | Medium | 6.1 | Task 1 |
| V06 | HTTP TRACE Method Enabled | Medium | 5.4 | Task 1 |
| V07 | Excessive Directory Exposure | Medium | 5.3 | Task 2 |
| V08 | CGI Directory Present and Confirmed | High | 7.3 | Task 2 |
| V09 | FrontPage Server Extensions Exposed | Medium | 4.8 | Task 2 |

**Total: 9 vulnerabilities. 2 Critical, 3 High, 4 Medium.**

---

## Recommendations

| Priority | Recommendation | Addresses |
|:---------|:---------------|:----------|
| Critical | Implement HTTPS with TLS 1.2 or higher and HSTS | V01, V02, V03 |
| High | Upgrade IIS 8.5 to IIS 10.0 on Windows Server 2022 | V04 |
| High | Set HttpOnly and Secure flags on all session cookies | V05 |
| Medium | Disable the HTTP TRACE method via web.config | V06 |
| High | Restrict and remove sensitive directories (CGI, FrontPage) | V07, V08, V09 |
| Medium | Implement Multi Factor Authentication (TOTP or SMS) | V01, V02 |
| Medium | Add login rate limiting and account lockout after 5 failures | V01, V02 |
| Low | Sanitise robots.txt and remove sensitive path disclosures | V07 |
| Medium | Conduct regular security assessments (annual penetration test and monthly scans) | All |

---

## Tools Used

| Tool | Version | Purpose |
|:-----|:--------|:--------|
| **Nmap** | 7.98 | Port scanning, service detection, OS fingerprinting, NSE scripts |
| **Gobuster** | 3.8.2 | Web directory directory enumeration |
| **FFUF** | 2.1.0 | Web fuzzing and cross validation of directory enumeration |
| **Wireshark** | Latest | Live packet capture, HTTP traffic analysis, credential extraction |
| **Firefox ESR** | 140.0 | Generating authentic HTTP traffic to the target |
| **Kali Linux** | 2024 | Primary testing environment (WSL2 on Windows 10) |

---

## Repository Structure

```
shadowfox cybersecurity internship
    README.md
    Arun_Kumar_ShadowFox_Beginner_Report_Final.pdf
```

---

## Final Statistics

| Metric | Value |
|:-------|:------|
| Tasks Completed | 3 of 3 |
| Vulnerabilities Found | 9 |
| Directories Discovered | 14 |
| Critical Findings | 2 |
| Credential Sets Captured | 2 |

---

## Learning Outcomes

* Practical proficiency with Nmap port scanning and NSE script interpretation
* Web directory enumeration using multiple tools and cross validation
* Live network traffic analysis and credential extraction with Wireshark
* Understanding the critical difference between URL encoding and encryption
* Professional security report writing with CVSS assessed findings
* Applied NIST SP 800 115 and OWASP Testing Guide v4 methodology
* Practical experience with the full reconnaissance, exploitation, and reporting lifecycle

---

## Report

The complete detailed report with screenshots, annotated terminal outputs, and full proof of concept evidence is available in this repository:

[`Arun_Kumar_ShadowFox_Beginner_Report_Final.pdf`](./Arun_Kumar_ShadowFox_Beginner_Report_Final.pdf)

---

## Legal and Ethical Notice

All testing activities documented in this repository were performed **exclusively** on `testasp.vulnweb.com`, an intentionally vulnerable application provided by **Acunetix** for security education and training purposes. This system is freely and legally accessible for penetration testing practice.

No real users, production systems, private networks, or unauthorized targets were accessed. This work was conducted strictly for **educational purposes** as part of the ShadowFox Cyber Security Internship Program.

---

## Contact

**Arun Kumar** | Cyber Security Intern | ShadowFox

| | |
|:--|:--|
| **Email** | arungaming1973@gmail.com |
| **University** | Himachal Pradesh Technical University |

---
