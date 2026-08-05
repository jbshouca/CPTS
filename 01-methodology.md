# 01 — Penetration Testing Methodology

This covers HTB Academy Modules 1 (Penetration Testing Process) and 2 (Getting Started). The CPTS exam simulates a real engagement from start to finish, so understanding the professional methodology isn't optional — it's how you structure your 10 days.

---

## The Penetration Testing Stages

Every professional engagement follows these stages. On the CPTS, you move through them organically as you progress deeper into the network.

```
1. PRE-ENGAGEMENT
   ↓
2. INFORMATION GATHERING
   ↓
3. VULNERABILITY ASSESSMENT
   ↓
4. EXPLOITATION
   ↓
5. POST-EXPLOITATION
   ↓
6. LATERAL MOVEMENT
   ↓
7. PROOF-OF-CONCEPT / DOCUMENTATION
   ↓
8. REPORTING
```

### Stage 1: Pre-Engagement

**What it is:** Everything that happens before you touch a keyboard. On a real engagement, this is contracts, scope, rules of engagement. On the CPTS exam, this is reading the letter of engagement and understanding the rules.

**On the CPTS exam specifically:**

```
READ THE ENGAGEMENT LETTER CAREFULLY:
  - What IP ranges are in scope?
  - What are you authorized to attack?
  - Are there any restrictions?
  - What flags do you need to submit?
  - What's the reporting format?
  - What's the timeline?

Document this FIRST, before you scan anything.
```

**Key concepts from the HTB module:**

```
Types of penetration tests:
  External     — attack from the internet against public-facing assets
  Internal     — attack from inside the corporate network
  Web app      — focused on web application vulnerabilities
  Network      — focused on infrastructure and services
  Physical     — test physical security controls
  Social eng.  — phishing, vishing, pretexting

  CPTS exam combines: external + internal + web app + network

Scoping:
  - IP addresses/ranges in scope
  - Domains and subdomains
  - Specific applications
  - Time window for testing
  - Rules of engagement (what you can/cannot do)
  - Emergency contacts
  - Data handling requirements

Rules of Engagement (ROE):
  - Can you use automated scanners?
  - Can you attempt denial of service?
  - Can you create user accounts?
  - Can you modify data?
  - What time is testing allowed?
  - Who do you contact if something breaks?
```

### Stage 2: Information Gathering

**What it is:** Discovering everything about the target — hosts, services, versions, web applications, users, network topology.

```
PASSIVE RECON (no direct interaction with target):
  - WHOIS lookups
  - DNS enumeration
  - Google dorking
  - Certificate transparency logs (crt.sh)
  - Shodan/Censys
  - Social media / LinkedIn

ACTIVE RECON (directly interacting with target):
  - Nmap scanning
  - Service enumeration
  - Web crawling and directory brute forcing
  - Banner grabbing
  - DNS zone transfers
```

On the CPTS exam, you'll do primarily active recon since you're given a target network.

### Stage 3: Vulnerability Assessment

**What it is:** Analyzing your enumeration findings to identify potential vulnerabilities.

```
For each service found:
  1. Identify the exact version
  2. Search for known CVEs (searchsploit, Google)
  3. Check for default credentials
  4. Check for misconfigurations
  5. Test for common vulnerability classes (SQLi, LFI, etc.)
  6. Assess the risk (how exploitable? what access does it give?)
```

### Stage 4: Exploitation

**What it is:** Using identified vulnerabilities to gain access.

```
Exploitation approach:
  1. Start with the highest-confidence, lowest-risk vulnerability
  2. Set up your listener/handler before executing
  3. Document every step (commands, output, screenshots)
  4. If exploitation fails, move to the next candidate
  5. After initial access, stabilize your shell immediately
```

### Stage 5: Post-Exploitation

**What it is:** Everything you do after gaining access to a system.

```
Immediate actions after getting a shell:
  1. Stabilize the shell (python PTY, PowerShell upgrade)
  2. whoami / id — who are you?
  3. hostname — where are you?
  4. ip a / ipconfig — what networks are connected?
  5. Look for flags (submit them immediately)
  6. Enumerate for privilege escalation
  7. Search for credentials (files, history, memory)
  8. Map the network (ARP table, routing table, internal DNS)
```

### Stage 6: Lateral Movement

**What it is:** Moving from one compromised system to another.

```
Lateral movement methods:
  - Credential reuse (found creds → try on every other service)
  - Pass the Hash (NTLM hash → psexec, evil-winrm)
  - SSH with found keys or passwords
  - Pivoting through tunnels to reach internal networks
  - Kerberos attacks (Kerberoasting, AS-REP roasting)
  - RDP, WinRM, SMB with found credentials

CRITICAL FOR CPTS: The exam network requires chaining across multiple hosts.
You compromise machine A, find creds for machine B, pivot to machine B,
find information leading to machine C, etc. It's a chain, not isolated targets.
```

### Stage 7: Documentation

**What it is:** Recording every finding with enough detail to reproduce it.

```
For EVERY vulnerability and exploitation:
  - Exact steps to reproduce (commands, URLs, payloads)
  - Screenshots of key moments (before/after exploitation)
  - Impact assessment (what can an attacker do with this access?)
  - Risk rating (Critical/High/Medium/Low/Informational)
  - Remediation recommendation
```

### Stage 8: Reporting

**What it is:** The commercial-grade report that determines whether you pass.

```
CPTS report structure:
  - Executive Summary (for non-technical readers)
  - Scope and Methodology
  - Findings (each vulnerability detailed)
  - Risk Assessment
  - Remediation Recommendations
  - Appendices (tool output, full scan results)

THE REPORT DETERMINES YOUR PASS/FAIL.
Even if you find all flags, a poor report can fail you.
Even if you miss some flags, an excellent report can pass you.
```

---

## CPTS Methodology — The Flow

On the exam, your 10 days should follow this general pattern:

```
Days 1-2: SCAN EVERYTHING
  - Full port scans on all in-scope ranges
  - Service enumeration on every open port
  - Web enumeration on every web server
  - Identify the network topology
  
Days 3-6: EXPLOIT AND PIVOT
  - Attack the easiest targets first
  - Harvest credentials from every compromised host
  - Try found credentials EVERYWHERE
  - Pivot to internal networks
  - Attack Active Directory
  
Days 7-8: DEEP DIVE
  - Revisit machines you couldn't fully exploit
  - Check for overlooked services or web directories
  - Try different attack vectors on stubborn targets
  - Ensure all flags are submitted
  
Days 9-10: CLEANUP NOTES
  - Organize all documentation
  - Verify all screenshots are present
  - Start drafting the report outline

Days 11-14 (REPORT WINDOW):
  - Write the full commercial-grade report
  - Review, proofread, format
  - Submit before the deadline
```

---

## Key Differences from OSCP Methodology

```
OSCP: Each machine is independent. Root machine A, move to machine B.
CPTS: Machine A gives you creds for machine B. Machine B has a web app
      that reveals info about machine C. Machine C has AD creds for
      the domain controller. It's a CHAIN.

OSCP: 24 hours — speed matters. Skip machines you're stuck on.
CPTS: 10 days — thoroughness matters. Enumerate EVERYTHING.

OSCP: Report is technical walkthrough. Adequate documentation passes.
CPTS: Report is commercial-grade. Professional quality required.
      Risk assessment, business impact, remediation — all mandatory.

OSCP: Submit proof.txt hashes as evidence.
CPTS: Submit flags through the HTB platform during the exam.
```
