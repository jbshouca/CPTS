# CPTS Practice Labs

Hands-on labs for HTB CPTS exam preparation. Each lab emphasizes CPTS-specific skills: vulnerability chaining, credential reuse, enterprise methodology, and commercial-grade reporting.

## Enterprise Simulation
| Lab | Lines | Skills Practiced |
|---|---|---|
| [Enterprise Attack Chain](lab-enterprise-chain.md) | 480 | Multi-machine credential chaining across 4 VMs — simulates the CPTS exam |

## CPTS-Specific Web Attacks
| Lab | Lines | Skills Practiced |
|---|---|---|
| [IDOR, XXE, Verb Tampering](lab-idor-xxe-verb.md) | 460 | Three CPTS-specific web attack classes not on OSCP |
| [ffuf Practical](lab-ffuf-practical.md) | 405 | 5 exercises: directory, vhost, parameter, POST, combined |
| [SQLMap Practical](lab-sqlmap-practical.md) | 430 | GET/POST/cookie injection, file read, OS shell |
| [Web Recon Practical](lab-web-recon.md) | 340 | Source analysis, JS mining, backup files, API discovery |

## Application Exploitation
| Lab | Lines | Skills Practiced |
|---|---|---|
| [Attacking Applications](lab-attacking-applications.md) | 375 | Tomcat WAR upload, Jenkins script console, helpdesk mining |

## Infrastructure
| Lab | Lines | Skills Practiced |
|---|---|---|
| [Double Pivoting](lab-double-pivot.md) | 345 | 5 two-hop methods: SOCKS, jump, nested, port forward, reverse |
| [Service Enumeration](lab-service-enumeration.md) | 310 | SMB/FTP/NFS/DNS/SNMP — find creds in every service |

## Reporting
| Lab | Lines | Skills Practiced |
|---|---|---|
| [Report Writing](lab-report-writing.md) | 340 | Commercial-grade template with self-grading rubric |

## Cross-Reference: OSCP Labs Also Useful for CPTS

| OSCP Lab | Why It Helps for CPTS |
|---|---|
| Web Attacks Deep Dive | SQLi, LFI, command injection, file upload fundamentals |
| Build Your Own AD Lab | AD is the backbone of the CPTS exam |
| SSH Tunnel Drill | Pivoting is mandatory on CPTS |
| Burp Suite Practical | Web proxy for IDOR and XXE testing |
| Password Cracking | Kerberos hash cracking |
| Linux/Windows Privesc Playgrounds | Same vectors appear on CPTS |
| Exam Simulations 1-3 | Practice methodology under time pressure |
