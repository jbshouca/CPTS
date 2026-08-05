# CPTS Lab: Report Writing Practice

## Objective

Write a commercial-grade penetration test report based on the Enterprise Attack Chain lab. This is the skill that determines whether you pass or fail the CPTS — practice it before the exam.

---

## The Exercise

**After completing the Enterprise Attack Chain lab** (lab-enterprise-chain.md), write a full report as if you were delivering it to a real client. Use the template below. Set a timer for 4 hours — that's a realistic pace for a shorter engagement report.

---

## Report Template

Copy this structure into a document. Fill in every section.

### Cover Page

```
PENETRATION TEST REPORT

Client:         [Target Company Name]
Assessment:     Internal Network Penetration Test
Period:         [Date range]
Prepared by:    [Your Name]
Classification: CONFIDENTIAL

[Your company logo or name]
```

### Table of Contents

```
1. Executive Summary
2. Scope and Methodology
3. Summary of Findings
4. Detailed Findings
   4.1 [Finding Title]
   4.2 [Finding Title]
   ...
5. Attack Narrative
6. Remediation Summary
7. Appendices
```

### 1. Executive Summary

Write this for a CEO who doesn't understand technology. One page maximum.

```
TEMPLATE:

[Company] engaged [Your Name] to conduct an internal network penetration
test of the corporate environment during [dates]. The objective was to
identify vulnerabilities that could allow an unauthorized party to gain
access to sensitive systems and data.

During the assessment, [X] vulnerabilities were identified:
  - [N] Critical
  - [N] High
  - [N] Medium
  - [N] Low
  - [N] Informational

The assessment demonstrated that an attacker with [describe initial
position — e.g., "access to the corporate network"] could achieve
[describe worst outcome — e.g., "full administrative control of the
Active Directory domain"] within [timeframe].

The most significant findings include:
  1. [Finding 1 — plain English, business impact]
  2. [Finding 2 — plain English, business impact]
  3. [Finding 3 — plain English, business impact]

Immediate action is recommended for the [N] critical findings.
A prioritized remediation plan is provided in Section 6.

Positive observations:
  - [Something the client is doing well — e.g., "SSH was properly
    hardened on most Linux hosts"]
  - [Another positive observation]
```

### 2. Scope and Methodology

```
2.1 Scope

The following systems were in scope for this assessment:
  - 192.168.244.132 (Debian — Web application server)
  - 192.168.244.131 (CentOS — Jump box / middleware)
  - 172.16.0.2 (Ubuntu — Internal deployment server)
  - 192.168.244.XXX (Windows 11 — Monitoring workstation)

2.2 Methodology

The assessment followed the Penetration Testing Execution Standard (PTES)
and OWASP Testing Guide v4.2 for web application testing. Findings are
classified using the Common Vulnerability Scoring System (CVSS) v3.1.

Phases:
  1. Reconnaissance and enumeration
  2. Vulnerability identification
  3. Exploitation and post-exploitation
  4. Lateral movement and privilege escalation
  5. Documentation and reporting

2.3 Tools Used

  - Nmap 7.94 (network scanning)
  - ffuf 2.1.0 (web fuzzing)
  - Burp Suite Community Edition (web application testing)
  - SQLMap 1.8 (SQL injection exploitation)
  - CrackMapExec 5.4 (Windows/AD enumeration)
  - Impacket 0.12 (Windows protocol attacks)
  - John the Ripper 1.9 / Hashcat 6.2 (password cracking)
```

### 3. Summary of Findings

```
| # | Title | Severity | CVSS | Host |
|---|-------|----------|------|------|
| 1 | SQL Injection in Employee Portal | Critical | 9.8 | 192.168.244.132 |
| 2 | IDOR in Employee API | High | 7.5 | 192.168.244.132 |
| 3 | Credentials Stored in Plaintext | High | 7.2 | 192.168.244.131 |
| 4 | Writable Cron Script | High | 7.8 | 172.16.0.2 |
| 5 | Credential Reuse Across Systems | High | 8.1 | Multiple |
| 6 | Sudo Misconfiguration | Medium | 6.7 | 192.168.244.132 |
| 7 | Missing HTTP Security Headers | Low | 3.1 | 192.168.244.132 |
```

### 4. Detailed Findings

**Repeat this format for EVERY finding:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FINDING 1: SQL Injection in Employee Portal
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Severity:     Critical
CVSS Score:   9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)
Affected:     192.168.244.132 — /app/ (Employee Portal)
CWE:          CWE-89: Improper Neutralization of Special Elements
              used in an SQL Command

DESCRIPTION:
The employee portal at /app/ accepts a user_id parameter that is
inserted directly into a SQL query without sanitization. This allows
an unauthenticated attacker to extract the entire database contents,
read files from the server filesystem, and potentially execute
operating system commands.

IMPACT:
An attacker can extract all employee records including plaintext
passwords, administrative credentials, and internal server
configurations. The extracted credentials were used to gain SSH access
to additional systems in the network (see Finding 5). This
vulnerability alone provided the initial foothold for the complete
compromise of the domain.

STEPS TO REPRODUCE:

Step 1: Navigate to http://192.168.244.132/app/?user_id=1
[SCREENSHOT: Normal product page]

Step 2: Inject a single quote to confirm SQL injection:
  http://192.168.244.132/app/?user_id=1'
[SCREENSHOT: SQL error message confirming injection]

Step 3: Determine the number of columns using ORDER BY:
  http://192.168.244.132/app/?user_id=1 ORDER BY 4 → success
  http://192.168.244.132/app/?user_id=1 ORDER BY 5 → error
  → 4 columns in the query

Step 4: Extract all credentials using UNION injection:
  http://192.168.244.132/app/?user_id=-1 UNION SELECT 1,
  GROUP_CONCAT(username,':',password),3,4 FROM users
[SCREENSHOT: All usernames and passwords displayed]

Step 5: Verify extracted credentials provide SSH access:
  ssh svc_centos@192.168.244.131 with password C3nt0sJump!
[SCREENSHOT: Successful SSH login]

REMEDIATION:
1. Implement parameterized queries (prepared statements) for all
   database interactions:

   VULNERABLE CODE:
   $r = $conn->query("SELECT * FROM users WHERE id=$id");

   REMEDIATED CODE:
   $stmt = $conn->prepare("SELECT * FROM users WHERE id = ?");
   $stmt->bind_param("i", $id);
   $stmt->execute();

2. Implement input validation to reject non-numeric input for the
   user_id parameter.
3. Apply the principle of least privilege to the database user —
   remove FILE privilege and restrict to SELECT-only access.
4. Deploy a Web Application Firewall (WAF) as defense in depth.

REFERENCES:
  - OWASP SQL Injection Prevention Cheat Sheet
  - CWE-89: https://cwe.mitre.org/data/definitions/89.html
```

### 5. Attack Narrative

```
TEMPLATE:

The engagement began with network enumeration of the in-scope IP range.
An HTML comment in the main page source code revealed an employee portal
at the /app/ path on 192.168.244.132.

Testing the employee portal's user_id parameter revealed a SQL injection
vulnerability (Finding 1). Through this vulnerability, the tester extracted
the complete users table, which contained plaintext credentials for four
accounts. Notably, the svc_centos account's notes field contained SSH
credentials for the CentOS jump box at 192.168.244.131.

Using the extracted credentials, the tester gained SSH access to the
CentOS server. Enumeration of the jumper user's home directory revealed
a deployment configuration file containing SSH credentials for the
internal Ubuntu server at 172.16.0.2 (Finding 3).

An SSH tunnel through the CentOS server provided access to the internal
network segment (172.16.0.0/24). The tester connected to the Ubuntu
server using the discovered credentials. On this server, a configuration
file contained RDP credentials for the Windows monitoring host and a
devops account password for the Debian server (Finding 5 — credential reuse).

The Windows monitoring host was accessed via RDP, and the Debian server
was accessed via SSH using the devops account. The devops account had
unrestricted sudo access (Finding 6), granting root privileges on the
Debian server and completing the attack chain.

Total time from unauthenticated access to full compromise: [X] hours.

The attack chain demonstrates how a single web vulnerability (SQL
injection), combined with credential storage issues and credential
reuse, can lead to the complete compromise of an enterprise network
spanning multiple network segments.
```

### 6. Remediation Summary

```
PRIORITY 1 — IMMEDIATE (within 1 week):
  □ Remediate SQL injection in employee portal (Finding 1)
  □ Remove plaintext credentials from configuration files (Finding 3)
  □ Change all compromised passwords across all systems

PRIORITY 2 — SHORT TERM (within 1 month):
  □ Implement access controls on the employee API (Finding 2)
  □ Fix writable cron script permissions (Finding 4)
  □ Implement password policy prohibiting credential reuse (Finding 5)
  □ Review and restrict sudo configurations (Finding 6)

PRIORITY 3 — LONG TERM (within 3 months):
  □ Implement network segmentation between server tiers
  □ Deploy a Web Application Firewall
  □ Implement a secrets management solution
  □ Add HTTP security headers (Finding 7)
  □ Conduct security awareness training on credential management
```

### 7. Appendices

```
Appendix A: Full Nmap Scan Results
  [Paste complete nmap output for all hosts]

Appendix B: SQLMap Output
  [Paste relevant SQLMap extraction output]

Appendix C: Discovered Credentials
  [Table of all credentials found, where found, and where they worked]

Appendix D: Flags Captured
  [List of all flags submitted during the assessment]
```

---

## Grading Yourself

After writing the report, check against this rubric:

```
EXECUTIVE SUMMARY:
  □ Non-technical — a CEO could understand it?
  □ Includes number and severity of findings?
  □ States the worst-case outcome?
  □ Lists top 3 recommendations?
  □ Includes positive observations?
  □ One page or less?

FINDINGS:
  □ Every finding has: title, severity, CVSS, description, impact,
    reproduction steps, evidence, remediation?
  □ Reproduction steps are exact enough for someone else to follow?
  □ Screenshots are present and readable?
  □ Remediation is specific and actionable (not just "fix it")?
  □ CVSS scores are justified?

ATTACK NARRATIVE:
  □ Tells a coherent story from start to finish?
  □ Shows how findings chain together?
  □ Demonstrates the compounding impact?

PRESENTATION:
  □ Professional formatting?
  □ No typos or grammatical errors?
  □ Consistent heading styles?
  □ Table of contents matches actual sections?
  □ Screenshots are labeled and referenced in text?
```
