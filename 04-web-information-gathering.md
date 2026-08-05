# 04 — Web Information Gathering

This covers the HTB Academy "Information Gathering — Web Edition" module. On the CPTS, web applications are the most common initial foothold. Thorough web enumeration — beyond just running ffuf — determines whether you find the attack surface.

---

## Passive Reconnaissance

Information you can gather WITHOUT directly interacting with the target.

### WHOIS

```bash
whois target.com
# Shows: registrant info, name servers, registration dates
# Useful for: finding related domains, admin contact info
```

### DNS enumeration

```bash
# Standard records
dig target.com A           # IPv4 addresses
dig target.com AAAA        # IPv6 addresses
dig target.com MX          # mail servers
dig target.com NS          # name servers
dig target.com TXT         # SPF records, verification strings, sometimes secrets
dig target.com ANY         # all records

# Subdomain discovery via certificate transparency
# crt.sh shows every SSL certificate ever issued for a domain
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sort -u
# Returns: mail.target.com, dev.target.com, vpn.target.com, etc.

# Zone transfer (get ALL records if the DNS server allows it)
dig axfr target.com @NS_SERVER
```

### Search engine discovery

```bash
# Google dorks
site:target.com                          # all indexed pages
site:target.com filetype:pdf             # PDF documents
site:target.com filetype:xlsx            # spreadsheets
site:target.com inurl:admin              # admin pages
site:target.com inurl:login              # login pages
site:target.com intitle:"index of"       # directory listings
site:target.com ext:php inurl:config     # PHP config files
"target.com" password                    # leaked credentials mentioning the domain
site:github.com "target.com"             # code mentioning the domain

# Wayback Machine (see old versions of the site)
# web.archive.org/web/*/target.com
# Old pages might have been removed but cached — credentials, admin panels, etc.
```

---

## Active Reconnaissance

### Technology identification

```bash
# Automated fingerprinting
whatweb http://TARGET
# Shows: web server, CMS, frameworks, JavaScript libraries, cookies

# Manual header check
curl -I http://TARGET
# Server: Apache/2.4.57 (Debian)
# X-Powered-By: PHP/8.1.2
# Set-Cookie: PHPSESSID=... (confirms PHP)
```

### Source code analysis

```bash
# Download and search the page source
curl -s http://TARGET > source.html

# Search for sensitive information
grep -iE "<!--.*-->" source.html          # HTML comments
grep -iE "password|secret|key|token|api" source.html
grep -iE "admin|internal|debug|todo|hack|fixme" source.html
grep -oE "https?://[^\"' >]+" source.html  # all URLs
grep -oE "/[a-zA-Z0-9_/-]+\.(php|asp|jsp)" source.html  # internal paths
grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" source.html  # emails
```

### JavaScript file analysis

Modern web apps load JavaScript files that reveal API endpoints, internal paths, and sometimes credentials:

```bash
# Find JavaScript files
curl -s http://TARGET | grep -oE 'src="[^"]*\.js"' | sed 's/src="//;s/"//'

# Download and search each JS file
curl -s http://TARGET/static/app.js | grep -iE "api|endpoint|url|password|key|token|secret"
curl -s http://TARGET/static/app.js | grep -oE "/api/[a-zA-Z0-9/_-]+"
# Found: /api/v1/users, /api/v1/admin/config, /api/internal/debug
# These are endpoints to test — they might not be linked from the UI
```

### Crawling and spidering

```bash
# wget spider (download the entire site structure)
wget --spider --force-html -r -l 3 http://TARGET 2>&1 | grep -oP "(?<=URL:)http[^ ]+"

# Or use Burp Suite's crawler
# Target → Site map → right-click domain → Spider this host
```

---

## Directory and File Enumeration

### ffuf (primary tool for CPTS)

```bash
# Standard directory scan
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e .php,.txt,.html,.bak,.conf,.xml,.json -fc 404

# Recursive
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.txt -recursion -recursion-depth 2 -fc 404

# Virtual host discovery
ffuf -u http://TARGET -H "Host: FUZZ.target.com" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs DEFAULT_SIZE
```

### Files to check manually

```bash
# Standard files
curl http://TARGET/robots.txt
curl http://TARGET/sitemap.xml
curl http://TARGET/.well-known/security.txt
curl http://TARGET/crossdomain.xml
curl http://TARGET/clientaccesspolicy.xml

# Version control
curl http://TARGET/.git/HEAD
curl http://TARGET/.svn/entries
curl http://TARGET/.hg/store/00manifest.i

# Environment and config
curl http://TARGET/.env
curl http://TARGET/.env.bak
curl http://TARGET/config.php.bak
curl http://TARGET/web.config
curl http://TARGET/wp-config.php.bak
curl http://TARGET/.htaccess
curl http://TARGET/.htpasswd

# Debug and info
curl http://TARGET/phpinfo.php
curl http://TARGET/info.php
curl http://TARGET/server-status
curl http://TARGET/server-info
curl http://TARGET/elmah.axd            # ASP.NET error log
curl http://TARGET/trace.axd            # ASP.NET trace

# Common backup patterns for any found file
# If you find /config.php, also check:
curl http://TARGET/config.php~
curl http://TARGET/config.php.bak
curl http://TARGET/config.php.old
curl http://TARGET/config.php.save
curl http://TARGET/config.php.swp
curl http://TARGET/.config.php.swp      # vim swap files have a dot prefix
curl http://TARGET/config.php.1
curl http://TARGET/config.copy.php
```

---

## Application Identification

When you find a web app, identify WHAT it is before attacking HOW:

```bash
# Check common CMS/app indicators
curl -s http://TARGET | grep -iE "wordpress|wp-content|drupal|joomla"
curl -s http://TARGET/wp-login.php            # WordPress
curl -s http://TARGET/administrator            # Joomla
curl -s http://TARGET/user/login               # Drupal
curl -s http://TARGET:8080/manager/html        # Tomcat
curl -s http://TARGET:8080/login               # Jenkins
curl -s http://TARGET/users/sign_in            # GitLab

# Check HTTP response headers
curl -I http://TARGET
# X-Powered-By: Express → Node.js
# X-Generator: Drupal → Drupal
# Server: Microsoft-IIS/10.0 → IIS on Windows

# Check cookies
# PHPSESSID → PHP
# JSESSIONID → Java
# ASP.NET_SessionId → ASP.NET
# connect.sid → Node.js Express
```

### After identifying the application

```
1. Check for default credentials (module 22 reference table)
2. Determine the exact version
3. searchsploit the application name + version
4. Google: "APPLICATION VERSION exploit CVE"
5. Check for application-specific configuration files
6. If CMS: run the appropriate scanner (wpscan, droopescan, joomscan)
```

---

## Web Enumeration Checklist for CPTS

```
FOR EVERY WEB SERVER (every port with HTTP/HTTPS):

PASSIVE:
  □ DNS records (A, AAAA, MX, TXT, NS)
  □ Certificate transparency (crt.sh)
  □ Google dorks (site:, filetype:, inurl:)

ACTIVE:
  □ whatweb fingerprinting
  □ curl -I (headers — server, x-powered-by, cookies)
  □ curl -s and grep for comments, paths, emails, JS files
  □ Analyze JavaScript files for API endpoints
  □ Check robots.txt, sitemap.xml
  □ Check .git/HEAD, .env, .htaccess, .htpasswd
  □ Check phpinfo.php, info.php, server-status
  □ Check backup files of any discovered pages (.bak, .old, ~, .swp)

FUZZING:
  □ ffuf directory scan with extensions (match technology)
  □ ffuf recursive scan on found directories
  □ ffuf virtual host scan
  □ ffuf parameter discovery on dynamic pages

APPLICATION:
  □ Identify the CMS/application
  □ Try default credentials
  □ Run application-specific scanner
  □ searchsploit the exact version
  □ Check for application-specific config paths

DOCUMENT:
  □ Every URL discovered
  □ Every parameter identified
  □ Every technology and version
  □ Every potential vulnerability
  □ Screenshots of interesting pages
```
