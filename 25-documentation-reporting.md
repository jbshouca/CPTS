# 25 — Documentation & Reporting

**The report determines whether you pass or fail CPTS.** This isn't an exaggeration — HTB exam reviewers have explicitly stated that the report quality matters as much as the flags. You can find every flag and still fail with a poor report. This module covers how to write a commercial-grade penetration test report.

---

## Why the CPTS Report Is Different from OSCP

```
OSCP report:
  - Technical walkthrough (step-by-step commands)
  - Proof screenshots (flag + whoami + ip)
  - Basic vulnerability description
  - Format: follow the OffSec template

CPTS report:
  - Executive summary for non-technical management
  - Risk assessment with severity ratings and CVSS
  - Business impact analysis
  - Detailed findings with reproduction steps
  - Attack narrative (the chain across the network)
  - Remediation recommendations (specific and actionable)
  - Professional formatting and presentation
  - Format: follow the HTB template EXACTLY
```

---

## Report Structure

### 1. Cover Page

```
Penetration Test Report
Client: [Target Organization Name]
Assessment Type: Internal/External Network Penetration Test
Date: [Start Date] — [End Date]
Prepared by: [Your Name]
Classification: Confidential
```

### 2. Table of Contents

Auto-generated from headings. Every finding should be listed.

### 3. Executive Summary (1-2 pages)

**Audience:** C-suite executives, managers, non-technical stakeholders. They want to know: how bad is it? What do we need to fix? How much will it cost us if we don't?

```
Structure:
  1. Purpose of the assessment (1-2 sentences)
  2. Scope (what was tested)
  3. Overall risk rating (Critical/High/Medium/Low)
  4. Key findings summary (numbers: X critical, Y high, Z medium)
  5. Top 3 recommendations (highest-impact fixes)
  6. Positive observations (what the client is doing right)

Example:
"[Company] engaged [Tester] to conduct an internal network penetration test 
against the corporate Active Directory environment. The assessment identified 
14 vulnerabilities, including 3 critical and 5 high-severity findings that, 
when chained together, allowed the tester to gain full domain administrator 
access from an unauthenticated external position within 48 hours.

The most impactful findings include a SQL injection vulnerability in the 
customer portal allowing database extraction, default credentials on the 
Jenkins build server enabling code execution, and a Kerberoastable service 
account with a weak password granting domain admin privileges.

Immediate remediation is recommended for the 3 critical findings. A full 
remediation plan is provided in the Findings section."
```

**What NOT to put in the executive summary:**
- Technical commands or tool output
- IP addresses or hostnames
- Jargon that requires security expertise to understand
- More than 2 pages

### 4. Scope and Methodology

```
Scope:
  - IP ranges tested: 10.10.10.0/24, 172.16.0.0/16
  - Applications: Customer Portal (portal.target.com), Jenkins (jenkins.internal)
  - Excluded: Production database servers, DMZ segment
  - Testing window: [dates]

Methodology:
  - OWASP Testing Guide v4.2 for web application testing
  - PTES (Penetration Testing Execution Standard) for network testing
  - MITRE ATT&CK framework for attack technique classification

Tools used:
  - Nmap for network discovery and service enumeration
  - Burp Suite for web application testing
  - BloodHound for Active Directory attack path analysis
  - Impacket for Windows protocol attacks
  - [list all major tools]
```

### 5. Findings (the bulk of the report)

**Each finding gets its own section with a consistent format:**

```
FINDING: [Clear descriptive title]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Severity:       Critical / High / Medium / Low / Informational
CVSS Score:     9.8 (explain the vector briefly)
Affected Host:  10.10.10.15 (portal.target.com)
Service:        HTTP (Apache/2.4.57, PHP/8.1.2)
CWE:            CWE-89: SQL Injection

DESCRIPTION:
[2-3 paragraphs explaining what the vulnerability is, in plain English.
 A non-security person should understand the concept after reading this.]

IMPACT:
[What can an attacker do with this vulnerability? Business impact, not just
 technical impact. "An attacker could extract the complete customer database
 including names, email addresses, and password hashes" is better than
 "UNION-based SQL injection allows data extraction."]

STEPS TO REPRODUCE:
[Exact step-by-step instructions with commands and screenshots.
 Someone following these steps should be able to reproduce the finding.]

  Step 1: Navigate to http://portal.target.com/search
  Step 2: Enter the following payload in the search field: [payload]
  [Screenshot showing the input]
  Step 3: Observe that the database version is returned in the response
  [Screenshot showing the output]
  Step 4: Extract database contents using: [command]
  [Screenshot showing extracted data]

EVIDENCE:
[Screenshots, command output, captured data — proof that it works]

REMEDIATION:
[Specific, actionable steps to fix the vulnerability. Not "fix the SQLi" but:]
  1. Implement parameterized queries (prepared statements) for all database
     interactions. Replace:
       $query = "SELECT * FROM users WHERE id = '$input'";
     With:
       $stmt = $conn->prepare("SELECT * FROM users WHERE id = ?");
       $stmt->bind_param("i", $input);
  2. Implement input validation — reject input containing SQL metacharacters
  3. Apply the principle of least privilege to the database user account
  4. Deploy a Web Application Firewall (WAF) as an additional layer of defense

REFERENCES:
  - OWASP SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
  - CWE-89: https://cwe.mitre.org/data/definitions/89.html
```

### 6. Attack Narrative

**This is unique to CPTS.** The attack narrative tells the story of how you progressed through the network. It shows how individual vulnerabilities chain together to create a much larger impact than any single finding alone.

```
Structure: chronological story of the engagement

Example:
"Initial external enumeration revealed a web application on port 443 
with a login form vulnerable to SQL injection (Finding #1). Through this 
vulnerability, the tester extracted database credentials including a 
plaintext password for the 'deploy' account.

Testing this password against SSH services across the network revealed 
credential reuse on the Jenkins build server at 10.10.10.20 (Finding #4). 
The Jenkins instance allowed access to the Groovy script console, enabling 
command execution on the underlying Linux server.

From the Jenkins server, internal network enumeration revealed an Active 
Directory domain (corp.target.com). Using the compromised Jenkins service 
account, the tester performed Kerberoasting and recovered the hash of the 
svc_sql service account (Finding #7). This hash was cracked offline, 
revealing a weak password.

The svc_sql account had local administrator privileges on the database 
server (10.10.10.30), allowing credential dumping via secretsdump 
(Finding #8). Among the dumped credentials was a Domain Admin account, 
which provided full control of the Active Directory domain (Finding #9).

Total time from unauthenticated external access to Domain Admin: 6 hours."
```

**Why the narrative matters:** It demonstrates that a "medium" SQL injection combined with "low" credential reuse combined with "medium" weak password policy creates a "critical" attack chain. Individual findings look manageable; the narrative shows the compounding effect.

### 7. Appendices

```
Appendix A: Full Nmap Scan Results
Appendix B: Vulnerability Scanner Output
Appendix C: BloodHound Attack Path Graphs
Appendix D: Tools and Versions Used
Appendix E: Flags/Objectives Completed
```

---

## Severity Ratings

### CVSS v3.1 Quick Reference

| Severity | CVSS Range | Criteria |
|---|---|---|
| Critical | 9.0 — 10.0 | Unauthenticated RCE, full domain compromise |
| High | 7.0 — 8.9 | Authenticated RCE, sensitive data exposure, privesc |
| Medium | 4.0 — 6.9 | Requires user interaction, limited impact, authenticated access needed |
| Low | 0.1 — 3.9 | Information disclosure with minimal impact |
| Informational | 0.0 | Best practice recommendations, no direct exploit |

### Common finding severities

```
CRITICAL:
  - Unauthenticated SQL injection with data extraction
  - Unauthenticated remote code execution (CVE exploits)
  - Domain Admin compromise
  - Default credentials on critical infrastructure

HIGH:
  - Authenticated RCE (Jenkins script console, Tomcat manager)
  - Credential dumping (SAM, secretsdump, mimikatz)
  - Kerberoasting with crackable passwords
  - SUID/sudo privilege escalation to root

MEDIUM:
  - Cross-Site Scripting (XSS)
  - IDOR with limited data exposure
  - Missing security headers
  - Outdated software with known but complex-to-exploit CVEs

LOW:
  - Information disclosure (version banners, directory listings)
  - Missing HTTP security headers (no direct exploit)
  - Verbose error messages

INFORMATIONAL:
  - TLS configuration improvements
  - Password policy recommendations
  - Network segmentation suggestions
```

---

## Documentation During the Exam

### Note-taking structure

Create this directory structure at the start:

```bash
mkdir -p ~/cpts/{scans,notes,loot,screenshots,report}
```

For each host, create a notes file:

```bash
# ~/cpts/notes/10.10.10.15.md

# 10.10.10.15 — portal.target.com

## Ports
- 22/tcp SSH OpenSSH 8.9
- 80/tcp HTTP Apache/2.4.57
- 443/tcp HTTPS Apache/2.4.57

## Web Application
- Customer portal at /
- Login form at /login.php
- Search function at /search.php (VULNERABLE — SQLi)

## Credentials Found
- admin:Adm1nP@ss! (from SQLi on search.php)
- deploy:D3pl0y! (from database dump)

## Flags
- Flag 1: [submitted]

## Screenshots
- 10.10.10.15_nmap.png
- 10.10.10.15_sqli_proof.png
- 10.10.10.15_data_dump.png
```

### Screenshot checklist

For every finding, capture screenshots of:

```
□ The vulnerability discovery (scan output, web page, etc.)
□ The exploitation step (the payload being sent)
□ The result (data extracted, shell received, access gained)
□ The impact (what you can now access/do)
□ Any flags or proof of compromise
```

**Screenshots must be:**
- Clear and readable (increase terminal font if needed)
- Show the full command AND the output
- Include timestamps or context (hostname visible)
- Annotated if necessary (highlight the important part)

---

## Report Writing Tips

```
1. FOLLOW THE HTB TEMPLATE EXACTLY
   The reviewers use a rubric. Deviating from the expected format
   means they can't find the information they're looking for.

2. WRITE THE EXECUTIVE SUMMARY LAST
   You need to know all your findings before you can summarize them.

3. EVERY FINDING NEEDS REMEDIATION
   "Fix the SQL injection" is not remediation.
   "Implement parameterized queries using PDO prepared statements" is.

4. THE ATTACK NARRATIVE IS NOT OPTIONAL
   It shows you understand how the pieces fit together.

5. PROOFREAD EVERYTHING
   Typos and formatting errors look unprofessional.
   Have someone else read it if possible.

6. DON'T SUBMIT AT THE LAST MINUTE
   Technical issues happen. Submit with at least 2 hours to spare.

7. SCREENSHOTS ARE EVIDENCE
   A finding without a screenshot is a claim without proof.
   When in doubt, take the screenshot.
```
