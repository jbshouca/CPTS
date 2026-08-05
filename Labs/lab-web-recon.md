# CPTS Lab: Web Information Gathering Practical

## Objective

Practice the full web reconnaissance workflow against a realistic web application on your Debian VM. Source code analysis, JavaScript mining, backup file discovery, and technology fingerprinting — the skills that reveal attack surface before you fire a single exploit.

---

## Setup (Debian — 192.168.244.132)

```bash
sudo apt update && sudo apt install apache2 php libapache2-mod-php -y
sudo systemctl enable --now apache2

# Build a realistic multi-page application
sudo mkdir -p /var/www/html/{assets/js,assets/css,includes,old,api/v1,.git}

# === Main site ===
cat << 'PHPEOF' | sudo tee /var/www/html/index.php
<?php
// Company portal - version 3.2.1
// Last updated by sarah.chen@company.local
?>
<html>
<head>
    <title>TechCorp Portal</title>
    <meta name="generator" content="TechCorp CMS 3.2.1">
    <link rel="stylesheet" href="/assets/css/style.css">
    <script src="/assets/js/app.js"></script>
    <script src="/assets/js/api-client.js"></script>
</head>
<body>
<h1>Welcome to TechCorp</h1>
<p>Employee portal for internal use.</p>
<!-- TODO: Remove debug endpoint before production deploy -->
<!-- Contact: admin@techcorp.local for access issues -->
<!-- Old portal still accessible at /old/ for migration period -->
<nav>
    <a href="/about.php">About</a> |
    <a href="/contact.php">Contact</a> |
    <a href="/login.php">Staff Login</a>
</nav>
<footer>
    <p>&copy; 2026 TechCorp | <a href="/privacy.php">Privacy</a></p>
    <!-- Server: web01.internal.techcorp.local -->
</footer>
</body>
</html>
PHPEOF

# === JavaScript files with hidden endpoints ===
cat << 'JSEOF' | sudo tee /var/www/html/assets/js/app.js
// TechCorp Portal v3.2.1
// Build: 2026-07-15

const APP_CONFIG = {
    version: "3.2.1",
    environment: "production",
    api_base: "/api/v1",
    debug_endpoint: "/api/v1/debug",
    admin_panel: "/management/dashboard",
    legacy_api: "/api/v0/users"
};

function checkAuth() {
    fetch('/api/v1/auth/verify', {
        headers: { 'X-API-Key': 'dev-key-placeholder' }
    }).then(r => r.json());
}

function getUsers() {
    // Admin only endpoint
    fetch('/api/v1/admin/users').then(r => r.json());
}

// Debug function - remove before production
function debugInfo() {
    fetch('/api/v1/debug?verbose=true').then(r => r.json());
}
JSEOF

cat << 'JSEOF' | sudo tee /var/www/html/assets/js/api-client.js
// API Client Library
const API = {
    baseUrl: '/api/v1',
    endpoints: {
        login: '/api/v1/auth/login',
        register: '/api/v1/auth/register',
        profile: '/api/v1/user/profile',
        settings: '/api/v1/user/settings',
        upload: '/api/v1/files/upload',
        download: '/api/v1/files/download',
        admin_config: '/api/v1/admin/config',
        admin_users: '/api/v1/admin/users',
        admin_logs: '/api/v1/admin/logs',
        health: '/api/v1/health',
        metrics: '/api/v1/metrics'
    },
    internalServices: {
        database: 'db.internal.techcorp.local:3306',
        cache: 'redis.internal.techcorp.local:6379',
        queue: 'rabbit.internal.techcorp.local:5672'
    }
};
JSEOF

# === robots.txt ===
cat << 'EOF' | sudo tee /var/www/html/robots.txt
User-agent: *
Disallow: /api/
Disallow: /management/
Disallow: /old/
Disallow: /includes/
Disallow: /backup/
Disallow: /.git/
Sitemap: http://techcorp.local/sitemap.xml
EOF

# === .git exposure ===
echo "ref: refs/heads/main" | sudo tee /var/www/html/.git/HEAD
echo "Initial commit - portal v3.2.1" | sudo tee /var/www/html/.git/description
cat << 'EOF' | sudo tee /var/www/html/.git/config
[core]
    repositoryformatversion = 0
[remote "origin"]
    url = git@gitlab.internal.techcorp.local:webteam/portal.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[user]
    email = sarah.chen@techcorp.local
EOF

# === Config backup ===
cat << 'EOF' | sudo tee /var/www/html/includes/config.php.bak
<?php
// Database configuration
$db_host = 'db.internal.techcorp.local';
$db_user = 'portal_app';
$db_pass = 'P0rt@lDB2026!';
$db_name = 'techcorp_portal';

// API configuration
$api_key = 'sk_prod_FLAG{config_backup_exposed}';
$jwt_secret = 'sup3rs3cr3t_jwt_k3y_ch@ng3m3';

// SMTP
$smtp_host = 'mail.techcorp.local';
$smtp_user = 'noreply@techcorp.local';
$smtp_pass = 'M@ilP@ss2026';

// Internal services
$redis_host = 'redis.internal.techcorp.local';
$deploy_key = '/opt/keys/deploy_rsa';
?>
EOF

# === Old portal ===
echo "<h1>Legacy Portal v2.0</h1><p>This portal is deprecated. Please use the new portal.</p><p><!-- Default admin: admin / Legacy@dmin! --></p>" | sudo tee /var/www/html/old/index.html

# === API endpoints ===
echo '{"status":"ok","version":"3.2.1","uptime":"142 days"}' | sudo tee /var/www/html/api/v1/health.json

cat << 'PHPEOF' | sudo tee /var/www/html/api/v1/health.php
<?php
header('Content-Type: application/json');
echo json_encode([
    'status' => 'ok',
    'version' => '3.2.1',
    'server' => gethostname(),
    'php_version' => phpversion(),
    'uptime' => '142 days'
]);
?>
PHPEOF

# === Management dashboard placeholder ===
sudo mkdir -p /var/www/html/management
echo '<html><body><h1>Management Dashboard</h1><p>Admin access required.</p><p>FLAG{admin_panel_discovered}</p></body></html>' | sudo tee /var/www/html/management/dashboard.html

# === Login page ===
cat << 'PHPEOF' | sudo tee /var/www/html/login.php
<?php
$msg = "";
if(isset($_POST['username'])) {
    $msg = "<p style='color:red'>Invalid credentials for user: " . htmlspecialchars($_POST['username']) . "</p>";
    // Log failed attempts
    error_log("Failed login attempt for: " . $_POST['username'] . " from " . $_SERVER['REMOTE_ADDR']);
}
?>
<html>
<head><title>Staff Login - TechCorp</title></head>
<body>
<h1>Staff Login</h1>
<form method="POST" action="/login.php">
<input name="username" placeholder="Username"><br><br>
<input type="password" name="password" placeholder="Password"><br><br>
<!-- form version 2.1 -->
<input type="hidden" name="portal_version" value="3.2.1">
<input type="hidden" name="redirect" value="/dashboard">
<button type="submit">Login</button>
</form>
<?php echo $msg; ?>
</body>
</html>
PHPEOF

sudo iptables -F && sudo iptables -P INPUT ACCEPT
echo "Setup complete."
```

---

## Walkthrough

### Phase 1: Technology Fingerprinting

```bash
# whatweb
whatweb http://192.168.244.132
# Shows: Apache, PHP, title, meta generator

# Headers
curl -I http://192.168.244.132
# Check: Server, X-Powered-By, Set-Cookie
```

### Phase 2: Source Code Analysis

```bash
# Download and analyze the main page
curl -s http://192.168.244.132/ > /tmp/source.html

# HTML comments (developers leave secrets here)
grep -oP '<!--.*?-->' /tmp/source.html
# Finds:
#   TODO: Remove debug endpoint
#   Contact: admin@techcorp.local
#   Old portal at /old/
#   Server: web01.internal.techcorp.local

# Meta tags
grep -i "generator\|version\|author" /tmp/source.html
# Finds: TechCorp CMS 3.2.1

# Email addresses
grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+" /tmp/source.html
# Finds: admin@techcorp.local

# Internal paths and links
grep -oE 'href="[^"]*"' /tmp/source.html
grep -oE 'src="[^"]*"' /tmp/source.html
# Finds: /assets/js/app.js, /assets/js/api-client.js, /about.php, /login.php
```

### Phase 3: JavaScript File Analysis

```bash
# Download JS files
curl -s http://192.168.244.132/assets/js/app.js > /tmp/app.js
curl -s http://192.168.244.132/assets/js/api-client.js > /tmp/api.js

# Search for endpoints
grep -oE '"/[a-zA-Z0-9/_-]+"' /tmp/app.js /tmp/api.js
# Finds:
#   /api/v1/debug
#   /management/dashboard
#   /api/v0/users
#   /api/v1/auth/login
#   /api/v1/admin/config
#   /api/v1/files/upload

# Search for internal hostnames
grep -oE '[a-zA-Z0-9.-]+\.techcorp\.local' /tmp/api.js
# Finds:
#   db.internal.techcorp.local:3306
#   redis.internal.techcorp.local:6379
#   rabbit.internal.techcorp.local:5672

# Search for API keys, secrets, tokens
grep -iE "key|secret|token|password|api" /tmp/app.js
# Finds: dev-key-placeholder, debug endpoint reference
```

**These JS files revealed 12+ API endpoints and 3 internal hostnames that you'd NEVER find through directory brute forcing alone.**

### Phase 4: Standard Files

```bash
# robots.txt
curl http://192.168.244.132/robots.txt
# Disallowed paths = attack surface:
#   /api/, /management/, /old/, /includes/, /backup/, /.git/

# .git exposure
curl http://192.168.244.132/.git/HEAD
# ref: refs/heads/main — git repo is exposed!

curl http://192.168.244.132/.git/config
# Reveals: gitlab.internal.techcorp.local (internal GitLab)
# Reveals: sarah.chen@techcorp.local (developer email)
```

### Phase 5: Backup and Config Files

```bash
# Check for config backups
curl http://192.168.244.132/includes/config.php.bak
# JACKPOT: Database credentials, API key, JWT secret, SMTP creds, deploy key path
# P0rt@lDB2026!, sk_prod_FLAG{config_backup_exposed}, JWT secret

# Old portal
curl http://192.168.244.132/old/
# HTML comment reveals: admin / Legacy@dmin!
```

### Phase 6: API Exploration

```bash
# Test each endpoint found in JavaScript
curl http://192.168.244.132/api/v1/health.php
# Shows: server hostname, PHP version

curl http://192.168.244.132/management/dashboard.html
# Shows: FLAG{admin_panel_discovered}

# Hidden form fields
curl -s http://192.168.244.132/login.php | grep "hidden"
# portal_version=3.2.1, redirect=/dashboard
```

### Phase 7: Directory Fuzzing (AFTER manual analysis)

```bash
ffuf -u http://192.168.244.132/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.txt,.html,.bak -fc 404
# Confirms everything found manually and may reveal more
```

---

## What You Gathered (Summary)

```
CREDENTIALS:
  admin / Legacy@dmin! (from old portal comment)
  portal_app / P0rt@lDB2026! (from config.php.bak)
  noreply@techcorp.local / M@ilP@ss2026 (SMTP)

SECRETS:
  API key: sk_prod_FLAG{config_backup_exposed}
  JWT secret: sup3rs3cr3t_jwt_k3y_ch@ng3m3
  Deploy key path: /opt/keys/deploy_rsa

INTERNAL HOSTS:
  db.internal.techcorp.local:3306
  redis.internal.techcorp.local:6379
  rabbit.internal.techcorp.local:5672
  gitlab.internal.techcorp.local
  mail.techcorp.local
  web01.internal.techcorp.local

API ENDPOINTS:
  /api/v1/debug, /api/v1/auth/login, /api/v1/admin/config
  /api/v1/admin/users, /api/v1/files/upload
  /management/dashboard

PEOPLE:
  admin@techcorp.local
  sarah.chen@techcorp.local

EVERY PIECE OF THIS feeds into the next step of the attack chain.
```

---

## Cleanup

```bash
sudo rm -rf /var/www/html/{assets,includes,old,api,.git,management}
sudo rm /var/www/html/{index,login,about,contact,privacy}.php 2>/dev/null
sudo rm /var/www/html/robots.txt
echo "<h1>It works!</h1>" | sudo tee /var/www/html/index.html
```
