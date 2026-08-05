# HTB Certified Penetration Testing Specialist (CPTS) Study Guide

## What is CPTS?

The HTB CPTS is a hands-on penetration testing certification from Hack The Box Academy. Unlike certifications with multiple-choice questions, CPTS requires you to compromise a full enterprise Active Directory network — web apps, Linux hosts, Windows hosts, pivoting, and domain controllers — then write a professional report. It's widely considered an OSCP alternative that emphasizes vulnerability chaining and real-world enterprise environments.

---

## Exam Structure

```
Format:          10-day hands-on practical engagement + 4-day report writing window
Environment:     Full enterprise AD network (multiple subnets, web apps, domain-joined hosts)
Objective:       Find and submit all flags across the network
Deliverable:     Commercial-grade penetration test report
Passing:         Submit all/most flags + acceptable report quality
Retake:          Available if you fail (additional voucher required)
Cost:            ~$210 USD exam voucher (after completing the path)
Prerequisite:    Must complete ALL 28 modules in the Penetration Tester path (100%)
```

### How CPTS differs from OSCP

| Aspect | OSCP | CPTS |
|---|---|---|
| Duration | 23h 45m + 24h report | 10 days + 4 days report |
| Environment | 3 standalones + 1 AD set (independent machines) | Full interconnected enterprise network |
| Focus | Individual machine exploitation | Vulnerability chaining across network |
| Approach | Each machine has its own attack path | Must chain findings from one machine to reach the next |
| AD emphasis | 40% of exam (1 AD set) | Heavily AD-focused (entire network is AD) |
| Web attacks | Surface-level on some machines | Deep web app exploitation (IDOR, XXE, verb tampering) |
| Applications | Generic web apps | Real-world apps (WordPress, Tomcat, Jenkins, GitLab, etc.) |
| Report | Technical walkthrough | Commercial-grade report with risk assessment |
| Metasploit | Limited to 1 machine | No restrictions stated |

### Key differences in what you need to know

```
CPTS covers but OSCP does not:
  - IDOR (Insecure Direct Object References)
  - XXE (XML External Entity injection)
  - HTTP Verb Tampering
  - SQLMap (automated SQL injection)
  - ffuf (web fuzzing — more versatile than gobuster for CPTS)
  - Attacking specific applications (WordPress, Tomcat, Jenkins, Drupal,
    Joomla, osTicket, GitLab, Splunk, PRTG, Nagios, and more)
  - Detailed vulnerability assessment methodology
  - Commercial-grade report writing with risk ratings

OSCP covers but CPTS does not emphasize as much:
  - Buffer overflows (removed from newer OSCP but was classic)
  - Standalone machine exploitation (CPTS is fully networked)
  - Chisel/Ligolo (CPTS focuses more on SSH and Metasploit pivoting)
```

---

## The 28 HTB Academy Modules (Required Path)

You must complete 100% of the Penetration Tester path before taking the exam. Here's how this guide maps to those modules:

### Pre-Engagement & Foundations (Modules 1-3)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 1 | Penetration Testing Process | `01-methodology.md` |
| 2 | Getting Started | `01-methodology.md` |
| 3 | Network Enumeration with Nmap | `02-nmap.md` |

### Reconnaissance & Enumeration (Modules 4-6)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 4 | Footprinting | `03-footprinting-services.md` |
| 5 | Information Gathering — Web Edition | `04-web-information-gathering.md` |
| 6 | Vulnerability Assessment | `05-vulnerability-assessment.md` |

### Infrastructure Exploitation (Modules 7-12)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 7 | File Transfers | `06-file-transfers.md` |
| 8 | Shells & Payloads | `07-shells-payloads.md` |
| 9 | Using the Metasploit Framework | `08-metasploit.md` |
| 10 | Password Attacks | `09-password-attacks.md` |
| 11 | Attacking Common Services | `10-attacking-services.md` |
| 12 | Pivoting, Tunneling, and Port Forwarding | `11-pivoting-tunneling.md` |

### Active Directory (Module 13)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 13 | Active Directory Enumeration & Attacks | `12-active-directory.md` |

### Web Application Attacks (Modules 14-23)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 14 | Using Web Proxies | `13-web-proxies.md` |
| 15 | Attacking Web Applications with Ffuf | `14-ffuf-web-fuzzing.md` |
| 16 | Login Brute Forcing | `09-password-attacks.md` |
| 17 | SQL Injection Fundamentals | `15-sql-injection.md` |
| 18 | SQLMap Essentials | `16-sqlmap.md` |
| 19 | Cross-Site Scripting (XSS) | `17-xss.md` |
| 20 | File Inclusion | `18-file-inclusion.md` |
| 21 | File Upload Attacks | `19-file-upload-attacks.md` |
| 22 | Command Injections | `20-command-injection.md` |
| 23 | Web Attacks (Verb Tampering, IDOR, XXE) | `21-web-attacks-advanced.md` |

### Application-Specific Attacks (Module 24)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 24 | Attacking Common Applications | `22-attacking-applications.md` |

### Privilege Escalation (Modules 25-26)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 25 | Linux Privilege Escalation | `23-linux-privesc.md` |
| 26 | Windows Privilege Escalation | `24-windows-privesc.md` |

### Reporting & Capstone (Modules 27-28)

| # | HTB Module | Guide Coverage |
|---|---|---|
| 27 | Documentation & Reporting | `25-documentation-reporting.md` |
| 28 | Attacking Enterprise Networks | `26-enterprise-attack-simulation.md` |

### Quick References

| File | What it is |
|---|---|
| `cheatsheet.md` | Print-ready quick reference for exam day |
| `exam-strategy.md` | 10-day exam time management and methodology |

---

## Critical Modules — Where to Focus

Based on exam reviews and pass reports, these modules are the most exam-relevant:

```
MOST CRITICAL (spend the most time here):
  1. Active Directory Enumeration & Attacks (36 sections — the biggest module)
  2. Attacking Common Applications (33 sections — real-world app exploitation)
  3. Attacking Enterprise Networks (14 sections — simulates the actual exam)
  4. Documentation & Reporting (8 sections — the report determines pass/fail)

HEAVILY TESTED:
  5. Pivoting, Tunneling, and Port Forwarding (the network is multi-subnet)
  6. Windows Privilege Escalation (33 sections — extensive coverage)
  7. Linux Privilege Escalation (28 sections)
  8. Password Attacks (26 sections — credential reuse is everywhere)

FOUNDATIONAL (must know cold):
  9. Footprinting (service enumeration is constant)
  10. SQL Injection + SQLMap (common initial foothold)
  11. File Inclusion / File Upload / Command Injection (standard web footholds)
  12. Web Attacks — IDOR, XXE, Verb Tampering (CPTS-specific, not on OSCP)
```

---

## Study Timeline

### If you're starting fresh (16-20 weeks)

```
Weeks 1-2:    Modules 1-3  (Methodology, Getting Started, Nmap)
Weeks 3-4:    Modules 4-6  (Footprinting, Web Recon, Vuln Assessment)
Weeks 5-6:    Modules 7-9  (File Transfers, Shells, Metasploit)
Weeks 7-8:    Modules 10-12 (Passwords, Services, Pivoting)
Weeks 9-11:   Module 13    (Active Directory — spend extra time here)
Weeks 12-13:  Modules 14-18 (Web Proxies, Ffuf, Login Brute, SQLi, SQLMap)
Weeks 14-15:  Modules 19-23 (XSS, LFI, Upload, CmdI, IDOR/XXE/Verb)
Week 16:      Module 24    (Attacking Common Applications)
Weeks 17-18:  Modules 25-26 (Linux + Windows Privilege Escalation)
Week 19:      Modules 27-28 (Reporting + Enterprise Networks capstone)
Week 20:      Review, practice ProLabs, take the exam
```

### If you already passed OSCP (6-8 weeks)

```
Weeks 1-2:    Skim modules 1-12 (review — most overlaps with OSCP)
              FOCUS on: Footprinting depth, SQLMap, Attacking Common Services
Weeks 3-4:    Modules 13 + 23 (AD deep dive + IDOR/XXE/Verb Tampering — CPTS-specific)
Weeks 5-6:    Module 24 (Attacking Common Applications — this is NEW for you)
Week 7:       Modules 27-28 (Reporting format + Enterprise Networks capstone)
Week 8:       Take the exam
```

---

## How to Use This Guide

```
1. Read the module in this guide
2. Complete the corresponding HTB Academy module (interactive labs)
3. Practice the skills in your home lab or HTB machines
4. Add notes and commands to your personal cheatsheet
5. Move to the next module

The guide complements HTB Academy — it doesn't replace it.
HTB Academy has interactive labs you can't get from a markdown file.
This guide explains the WHY behind each technique and provides
reference material you can search during the exam.
```
