# CPTS Lab: ffuf Practical — Web Fuzzing Mastery

## Objective

Practice every ffuf fuzzing mode against real applications on your Debian VM. By the end you'll be able to discover hidden directories, virtual hosts, parameters, and valid credentials using ffuf fluently.

---

## Setup (Debian — 192.168.244.132)

```bash
sudo apt update && sudo apt install apache2 php libapache2-mod-php -y
sudo systemctl enable --now apache2

# === Exercise 1: Directory/file discovery ===
sudo mkdir -p /var/www/html/{admin,backup,internal,api/v1,api/v2,.hidden}
echo "<h1>Admin Panel</h1><p>FLAG{directory_discovered}</p>" | sudo tee /var/www/html/admin/index.html
echo "DB_PASS=BackupP@ss2026!" | sudo tee /var/www/html/backup/config.bak
echo "<h1>Internal Tools</h1>" | sudo tee /var/www/html/internal/index.html
echo '{"status":"ok","version":"1.0"}' | sudo tee /var/www/html/api/v1/status.json
echo '{"status":"ok","version":"2.0","debug":true}' | sudo tee /var/www/html/api/v2/status.json
echo "FLAG{hidden_dotfolder}" | sudo tee /var/www/html/.hidden/secret.txt
echo '<?php echo "User ID: " . ($_GET["id"] ?? "none") . " | Debug: " . ($_GET["debug"] ?? "off"); ?>' | sudo tee /var/www/html/internal/user.php
echo "robots.txt contents:" | sudo tee /var/www/html/robots.txt
echo "Disallow: /internal/" | sudo tee -a /var/www/html/robots.txt
echo "Disallow: /api/" | sudo tee -a /var/www/html/robots.txt
echo "<h1>Company Website</h1>" | sudo tee /var/www/html/index.html

# === Exercise 2: Virtual host discovery ===
# Create vhost configs
cat << 'EOF' | sudo tee /etc/apache2/sites-available/dev.conf
<VirtualHost *:80>
    ServerName dev.company.local
    DocumentRoot /var/www/dev
</VirtualHost>
EOF

cat << 'EOF' | sudo tee /etc/apache2/sites-available/staging.conf
<VirtualHost *:80>
    ServerName staging.company.local
    DocumentRoot /var/www/staging
</VirtualHost>
EOF

cat << 'EOF' | sudo tee /etc/apache2/sites-available/admin-portal.conf
<VirtualHost *:80>
    ServerName admin.company.local
    DocumentRoot /var/www/admin-portal
</VirtualHost>
EOF

sudo mkdir -p /var/www/{dev,staging,admin-portal}
echo "<h1>Development Server</h1><p>Debug mode enabled. DB: dev_db / D3vP@ss!</p>" | sudo tee /var/www/dev/index.html
echo "<h1>Staging Environment</h1><p>Pre-production testing.</p>" | sudo tee /var/www/staging/index.html
echo "<h1>Admin Portal</h1><p>FLAG{vhost_admin_found}</p><p>SSH: admin_user / @dminSSH2026</p>" | sudo tee /var/www/admin-portal/index.html

sudo a2ensite dev.conf staging.conf admin-portal.conf
sudo systemctl reload apache2

# === Exercise 3: Parameter fuzzing ===
cat << 'PHPEOF' | sudo tee /var/www/html/search.php
<?php
$output = "";
if(isset($_GET['q'])) {
    $output = "<p>Search results for: " . htmlspecialchars($_GET['q']) . "</p>";
}
if(isset($_GET['debug']) && $_GET['debug'] === 'true') {
    $output .= "<p style='color:red'>DEBUG: DB_HOST=localhost, DB_USER=root, DB_PASS=r00tDBp@ss!</p>";
}
if(isset($_GET['admin']) && $_GET['admin'] === '1') {
    $output .= "<p style='color:blue'>ADMIN MODE: FLAG{hidden_parameter_found}</p>";
}
if(isset($_GET['source'])) {
    $output .= "<pre>" . htmlspecialchars(file_get_contents(__FILE__)) . "</pre>";
}
echo "<html><body><h1>Search</h1>";
echo "<form method='GET'><input name='q' placeholder='Search...'> <button>Go</button></form>";
echo $output;
echo "</body></html>";
?>
PHPEOF

# === Exercise 4: POST login brute force ===
cat << 'PHPEOF' | sudo tee /var/www/html/login.php
<?php
$msg = "";
if(isset($_POST['username']) && isset($_POST['password'])) {
    $valid_users = [
        'admin' => 'SuperSecret123',
        'manager' => 'Management1!',
        'operator' => 'Oper@te2026'
    ];
    $u = $_POST['username'];
    $p = $_POST['password'];
    if(isset($valid_users[$u]) && $valid_users[$u] === $p) {
        $msg = "<p style='color:green'>Welcome, $u! FLAG{login_brute_forced}</p>";
    } else {
        $msg = "<p style='color:red'>Invalid credentials</p>";
    }
}
?>
<html><body>
<h1>Staff Login</h1>
<form method="POST">
Username: <input name="username"><br><br>
Password: <input type="password" name="password"><br><br>
<button type="submit">Login</button>
</form>
<?php echo $msg; ?>
</body></html>
PHPEOF

sudo iptables -F && sudo iptables -P INPUT ACCEPT
echo "Setup complete."
```

---

## Exercise 1: Directory and File Discovery

### Basic scan

```bash
ffuf -u http://192.168.244.132/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

**You'll see tons of results.** Many are 200 OK responses from the default page. Note the response size of the default page — then filter it out.

### Filtered scan (the right way)

```bash
# First, check what size the "not found" or default response is
curl -s http://192.168.244.132/nonexistent_asdf | wc -c
# Note the character count — let's say it's 43

# Now filter that size out
ffuf -u http://192.168.244.132/FUZZ -w /usr/share/wordlists/dirb/common.txt -fs 43
```

**What `-fs 43` does:** Filters out (hides) any response that is exactly 43 bytes. Since the "not found" page is always 43 bytes, only REAL pages show up.

### Add extensions

```bash
ffuf -u http://192.168.244.132/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.txt,.bak,.html,.json -fs 43
```

**What you should find:**
- `/admin/` — admin panel with a flag
- `/backup/config.bak` — database password
- `/internal/` — internal tools
- `/api/` — API endpoints
- `/robots.txt` — disallow entries
- `/search.php` — search functionality
- `/login.php` — login form
- `/.hidden/` — hidden dotfolder (if the wordlist includes dotentries)

### Recursive scan

```bash
ffuf -u http://192.168.244.132/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .json,.php -fs 43 -recursion -recursion-depth 2
```

**What this does:** When ffuf finds `/api/`, it automatically scans `/api/FUZZ`. When it finds `/api/v1/`, it scans `/api/v1/FUZZ`. This discovers `/api/v1/status.json` and `/api/v2/status.json` without you needing to scan each subdirectory manually.

### Challenge: Find the hidden dotfolder

```bash
# Most wordlists don't include entries starting with dots
# Use a wordlist that does, or create one
ffuf -u http://192.168.244.132/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -e .txt -fs 43

# Or target dotfolders specifically
echo -e ".hidden\n.secret\n.backup\n.config\n.git\n.env\n.svn\n.htpasswd\n.ssh" > /tmp/dotfiles.txt
ffuf -u http://192.168.244.132/FUZZ -w /tmp/dotfiles.txt -fs 43
# Then scan inside found folders
ffuf -u http://192.168.244.132/.hidden/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .txt -fs 43
```

---

## Exercise 2: Virtual Host Discovery

### Add the base domain to /etc/hosts first

```bash
echo "192.168.244.132 company.local" | sudo tee -a /etc/hosts
```

### Discover virtual hosts

```bash
# First check the default response size
curl -s -H "Host: nonexistent.company.local" http://192.168.244.132 | wc -c
# Note the size — this is what non-existent vhosts return

# Fuzz for valid vhosts
ffuf -u http://192.168.244.132 -H "Host: FUZZ.company.local" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs DEFAULT_SIZE
```

**What you should find:**
- `dev.company.local` — development server with DB credentials
- `staging.company.local` — staging environment
- `admin.company.local` — admin portal with SSH creds and a flag

### Access the discovered vhosts

```bash
# Add them to /etc/hosts
echo "192.168.244.132 dev.company.local staging.company.local admin.company.local" | sudo tee -a /etc/hosts

# Browse each one
curl http://dev.company.local
# Shows: Debug mode enabled. DB: dev_db / D3vP@ss!

curl http://admin.company.local
# Shows: FLAG{vhost_admin_found} and SSH creds
```

**Why this matters for CPTS:** Enterprise web servers often host multiple applications on different virtual hosts. If you only browse by IP, you see the default site and miss everything else. Virtual host fuzzing reveals hidden attack surfaces.

---

## Exercise 3: Parameter Fuzzing

### Discover hidden GET parameters

```bash
# Check the default response size (no parameters)
curl -s "http://192.168.244.132/search.php" | wc -c

# Fuzz parameter names
ffuf -u "http://192.168.244.132/search.php?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs DEFAULT_SIZE
```

**What you should find:**
- `q` — the search parameter (expected)
- `debug` — hidden debug mode
- `admin` — hidden admin mode
- `source` — source code disclosure

### Exploit found parameters

```bash
# Enable debug mode
curl "http://192.168.244.132/search.php?debug=true"
# Shows: DB credentials in debug output

# Enable admin mode
curl "http://192.168.244.132/search.php?admin=1"
# Shows: FLAG{hidden_parameter_found}

# Read source code
curl "http://192.168.244.132/search.php?source=1"
# Shows: PHP source code revealing all hidden parameters
```

### Fuzz parameter values

```bash
# What values does "debug" accept?
ffuf -u "http://192.168.244.132/search.php?debug=FUZZ" -w <(echo -e "true\nfalse\n1\n0\nyes\nno\non\noff") -fs DEFAULT_SIZE

# Fuzz user IDs on the internal page
ffuf -u "http://192.168.244.132/internal/user.php?id=FUZZ" -w <(seq 1 100) -fs DEFAULT_SIZE
```

---

## Exercise 4: Login Brute Force with ffuf

### Determine the failure response

```bash
curl -X POST http://192.168.244.132/login.php -d "username=admin&password=wrong"
# Response contains: "Invalid credentials"
# Note the response size for failed login
curl -s -X POST http://192.168.244.132/login.php -d "username=admin&password=wrong" | wc -c
# Say it's 350
```

### Brute force the password for a known user

```bash
ffuf -u http://192.168.244.132/login.php -X POST \
  -d "username=admin&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -w /usr/share/seclists/Passwords/Common-Credentials/top-1000.txt \
  -fs 350
```

**What this does:** Tries each password from the wordlist for the `admin` user. Failed logins return 350 bytes. A successful login returns a different size (it says "Welcome" instead of "Invalid"), so `-fs 350` hides failures and only shows the success.

### Brute force with username AND password lists

```bash
# Create a small user list
echo -e "admin\nmanager\noperator\nroot\ntest" > /tmp/users.txt

ffuf -u http://192.168.244.132/login.php -X POST \
  -d "username=FUZZU&password=FUZZP" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -w /tmp/users.txt:FUZZU \
  -w /usr/share/seclists/Passwords/Common-Credentials/top-1000.txt:FUZZP \
  -fs 350
```

**Two wordlists, two FUZZ positions:** `FUZZU` gets usernames, `FUZZP` gets passwords. ffuf tries every combination.

### Match on response content instead of size

```bash
# Instead of filtering by size, match responses containing "Welcome"
ffuf -u http://192.168.244.132/login.php -X POST \
  -d "username=admin&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -w /usr/share/seclists/Passwords/Common-Credentials/top-1000.txt \
  -mr "Welcome"
```

**`-mr "Welcome"`** = match responses containing "Welcome" — show only these. This is more reliable than size filtering when response sizes vary.

---

## Exercise 5: Combine Everything

Run through the full CPTS web enumeration workflow:

```bash
# 1. Directory scan with extensions
ffuf -u http://192.168.244.132/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.txt,.bak,.conf -fs 43

# 2. Virtual host scan
ffuf -u http://192.168.244.132 -H "Host: FUZZ.company.local" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs DEFAULT_SIZE

# 3. Parameter discovery on each page found
ffuf -u "http://192.168.244.132/search.php?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs DEFAULT_SIZE

# 4. Brute force any login forms found
ffuf -u http://192.168.244.132/login.php -X POST -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -w /usr/share/seclists/Passwords/Common-Credentials/top-1000.txt -fs 350

# 5. Recursive scan on interesting directories
ffuf -u http://192.168.244.132/api/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .json -fs 43 -recursion -recursion-depth 2
```

**Time yourself:** Complete all 5 steps in under 15 minutes.

---

## Cleanup

```bash
sudo rm -rf /var/www/html/{admin,backup,internal,api,.hidden,search.php,login.php}
sudo rm -rf /var/www/{dev,staging,admin-portal}
sudo a2dissite dev.conf staging.conf admin-portal.conf 2>/dev/null
sudo rm /etc/apache2/sites-available/{dev,staging,admin-portal}.conf
sudo systemctl reload apache2
sudo sed -i '/company.local/d' /etc/hosts    # on Kali
```
