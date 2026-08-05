# CPTS Exam Strategy — Managing 10 Days

The CPTS exam gives you 10 days to compromise an enterprise network plus 4 additional days to write your report. This sounds like a lot of time, but exam reviews consistently say it's needed — the network is large, complex, and requires methodical chaining. This module covers time management, methodology under pressure, and the report that determines your grade.

---

## The Exam Environment

```
What you're facing:
  - A full enterprise Active Directory network
  - Multiple subnets (external + internal + possibly more)
  - Web applications on multiple hosts
  - Domain-joined Windows machines
  - Linux servers with various services
  - Firewalls and segmentation between subnets
  - Vulnerability chaining required (machine A → machine B → machine C)
  - Flags hidden throughout the network (submit through HTB platform)

What you CANNOT do:
  - Attack infrastructure outside the stated scope
  - Perform denial-of-service attacks
  - Use commercial tools not available to you (Cobalt Strike, etc.)
  - Collaborate with others or share exam details

What you CAN do:
  - Use any tools available on a standard Kali installation
  - Use Metasploit without restriction
  - Reference your notes, cheatsheets, and external resources
  - Take breaks, sleep, come back refreshed
```

---

## Day-by-Day Strategy

### Day 1: Full Enumeration (8-10 hours of active work)

```
MORNING (3-4 hours):
  □ Read the letter of engagement completely
  □ Note all in-scope IPs and ranges
  □ Set up your directory structure:
    mkdir -p ~/cpts/{scans,notes,loot,screenshots,report}
  □ Launch full nmap scans on EVERYTHING:
    nmap -sV -sC -oA ~/cpts/scans/initial SCOPE_RANGE
    nmap -p- --min-rate=1000 -oA ~/cpts/scans/full SCOPE_RANGE
    sudo nmap -sU --top-ports 50 -oA ~/cpts/scans/udp SCOPE_RANGE
  □ While scans run, review quick scan results as they come in
  □ Start building a network map (which hosts, which ports)

AFTERNOON (3-4 hours):
  □ Enumerate EVERY web server found:
    - whatweb on each
    - Check source code, robots.txt, .git
    - ffuf directory brute force with extensions
    - Identify CMSs and applications
  □ Enumerate EVERY service found:
    - SMB shares (null session, guest access)
    - FTP (anonymous access)
    - SNMP (community string brute force)
    - DNS (zone transfer)
    - RDP, WinRM, SSH (note for later credential testing)
  □ Document everything in your notes

EVENING (2 hours):
  □ Review all findings
  □ Identify the most promising attack vectors
  □ Plan tomorrow's exploitation targets
  □ Organize notes into per-host files
```

### Day 2: First Footholds (8-10 hours)

```
FOCUS: Get initial access on the easiest targets

PRIORITY ORDER:
  1. Default credentials on web applications
     (admin:admin, admin:password, tomcat:tomcat, etc.)
  2. Known CVEs for identified software versions
  3. SQL injection on web forms
  4. Command injection on diagnostic tools
  5. File upload to get a web shell
  6. LFI to read config files with credentials
  7. IDOR / XXE / verb tampering on APIs

For EACH foothold:
  □ Stabilize the shell
  □ whoami, id, hostname, ip a
  □ Submit any flags found
  □ Screenshot everything
  □ Enumerate the host for credentials and pivoting info
  □ Check for privilege escalation vectors
  □ Document the full attack chain
```

### Days 3-4: Privilege Escalation and Credential Harvesting

```
For EVERY compromised host:
  □ Run enumeration scripts (LinPEAS, WinPEAS)
  □ Check sudo -l (Linux) / whoami /priv (Windows)
  □ Check for SUID binaries / service misconfigurations
  □ Search for credentials in files, history, config, registry
  □ Dump any accessible hashes
  □ Check network connections (what else can this host reach?)
  □ Check ARP table and routing table (discover internal hosts)

CREDENTIAL REUSE — THE MOST IMPORTANT STEP:
  Every credential you find, try on EVERY service:
  □ SSH on every Linux host
  □ RDP/WinRM on every Windows host
  □ SMB on every Windows host
  □ Web application logins
  □ Database logins
  □ FTP logins
  
  Use CrackMapExec to spray credentials across the network:
  crackmapexec smb RANGE -u found_user -p found_pass
  crackmapexec winrm RANGE -u found_user -p found_pass
```

### Days 5-7: Active Directory and Lateral Movement

```
AD ATTACK CHAIN:
  □ Enumerate the domain (users, groups, computers, shares)
  □ BloodHound data collection and analysis
  □ AS-REP Roasting (users without pre-auth)
  □ Kerberoasting (service accounts with SPNs)
  □ Password spraying with found/common passwords
  □ Check for credential reuse from previously found passwords
  □ Enumerate shares for sensitive documents
  □ Look for Group Policy Preferences (GPP) passwords
  □ Check for LAPS passwords
  □ Lateral movement with found credentials (PtH, WinRM, RDP)
  □ Escalate to Domain Admin
  □ DCSync for all domain hashes

PIVOTING:
  □ Set up SSH tunnels or SOCKS proxies to reach internal subnets
  □ Use Metasploit autoroute if you have Meterpreter sessions
  □ Scan internal networks through your tunnels
  □ Enumerate and exploit internal services
```

### Days 8-9: Cleanup and Missed Targets

```
REVISIT EVERYTHING:
  □ Go back to hosts you partially compromised
  □ Re-enumerate web applications with bigger wordlists
  □ Try different web attack vectors you haven't tested
  □ Check for hosts you discovered internally but haven't scanned
  □ Try password mutations on found credentials
  □ Read through ALL your notes — did you miss any leads?
  □ Verify all flags are submitted
  □ Take final screenshots with proof (flag + user + IP)
```

### Day 10: Documentation Prep

```
□ Organize all notes chronologically per host
□ Verify every screenshot is clear and properly labeled
□ Draft the report outline
□ Identify all findings to include (with severity ratings)
□ Start writing the executive summary
```

### Days 11-14: Report Writing

```
Day 11: Write the full report
Day 12: Add all screenshots and evidence
Day 13: Review, proofread, format
Day 14: Final review and submit (DON'T wait until the last hour)
```

---

## The "I'm Stuck" Protocol

```
Stuck for 1 hour:
  → Re-enumerate the current host (bigger wordlist, different extensions)
  → Check for services you haven't tested
  → Review your notes for leads you haven't followed

Stuck for 2 hours:
  → Move to a different host entirely
  → Try found credentials on hosts you haven't tried
  → Run a subnet scan from your current position
  → Check if you missed any web directories or hidden pages

Stuck for 4+ hours:
  → Take a real break (eat, walk, sleep)
  → Come back with fresh eyes
  → Re-read the engagement letter — did you miss something in scope?
  → Try the simplest things: default creds, obvious misconfigs
  → Check if there's a host you discovered but never fully enumerated

REMEMBER: You have 10 days. There is no reason to panic at hour 12.
Sleep. Eat. Come back sharp.
```

---

## Report — What Determines Pass/Fail

The CPTS report must be commercial-grade — the kind you'd deliver to a client.

### Report Structure

```
1. EXECUTIVE SUMMARY (1-2 pages)
   - Non-technical overview for management
   - Number and severity of findings
   - Overall risk level
   - Key recommendations (top 3)

2. SCOPE AND METHODOLOGY
   - What was tested (IPs, ranges, applications)
   - Testing methodology used (reference OWASP, PTES, etc.)
   - Tools used
   - Timeline of testing

3. FINDINGS (one section per vulnerability)
   For EACH finding:
   - Title (clear, descriptive)
   - Severity (Critical/High/Medium/Low/Informational)
   - CVSS score
   - Affected host(s) and service(s)
   - Description (what the vulnerability is)
   - Impact (what an attacker could do)
   - Steps to reproduce (exact commands + screenshots)
   - Remediation (specific, actionable steps to fix it)

4. ATTACK NARRATIVE
   - Chronological description of the attack chain
   - How vulnerability A led to access B led to escalation C
   - This shows the real-world impact of chaining vulnerabilities

5. APPENDICES
   - Full nmap scan output
   - Tool output
   - Additional screenshots
```

### Report Tips from People Who Passed

```
- Follow the HTB reporting template EXACTLY — don't improvise the format
- Every finding needs: description, impact, reproduction steps, remediation
- Screenshots must be clear and annotated
- The executive summary must be understandable by someone non-technical
- Risk ratings must be justified (explain WHY something is Critical vs High)
- Credential reuse across the network is a finding, not just a technique you used
- Missing patches are findings — document them with CVE numbers
- Weak password policy is a finding — include the policy you discovered
- The attack narrative should read like a story: "First we discovered X,
  which led us to Y, which gave us access to Z"
```

---

## Mental Model

```
OSCP mindset:  "How do I root this box?"
CPTS mindset:  "How does this finding connect to the next target?"

OSCP success:  Root each machine independently
CPTS success:  Chain findings across the entire network + write a great report

On CPTS, a password found on machine A isn't just a privesc —
it's potentially the foothold for machines B, C, and D.
Think LATERALLY, not just VERTICALLY.
```
