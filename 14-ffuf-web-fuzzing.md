# 14 — Web Fuzzing with ffuf

## What ffuf Is

ffuf (Fuzz Faster U Fool) is a web fuzzer written in Go. While gobuster brute forces directory names from a wordlist, ffuf is more flexible — it can fuzz ANY part of an HTTP request: URLs, parameters, headers, POST data, cookies. The CPTS path uses ffuf as the primary web enumeration tool instead of gobuster.

**Why ffuf over gobuster for CPTS:**
- Can fuzz URL paths, GET parameters, POST data, headers, cookies
- Better filtering (by status code, size, word count, line count)
- Supports multiple wordlist positions simultaneously
- Faster with recursive scanning
- More flexible output formatting

---

## Basic Directory Discovery

```bash
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

**Breaking this down:**

```
ffuf                    The tool
-u http://TARGET/FUZZ   The URL with FUZZ as the placeholder
                        ffuf replaces FUZZ with each word from the wordlist
-w wordlist.txt         The wordlist to use
```

**The keyword `FUZZ`** is the placeholder. ffuf replaces it with each word from the wordlist:

```
Word "admin"   → http://TARGET/admin
Word "backup"  → http://TARGET/backup
Word "config"  → http://TARGET/config
Word "uploads" → http://TARGET/uploads
```

### Adding file extensions

```bash
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.txt,.html,.bak
```

**What `-e` does:** Appends each extension to every word. The word "config" becomes:

```
http://TARGET/config
http://TARGET/config.php
http://TARGET/config.txt
http://TARGET/config.html
http://TARGET/config.bak
```

### Filtering results

ffuf can produce noisy output. Filter out the junk:

```bash
# Filter by status code (show only these)
ffuf -u http://TARGET/FUZZ -w wordlist.txt -mc 200,301,302,403

# Filter by status code (hide these)
ffuf -u http://TARGET/FUZZ -w wordlist.txt -fc 404

# Filter by response size (hide responses of this size)
ffuf -u http://TARGET/FUZZ -w wordlist.txt -fs 1234
# Use this when 404 pages return a custom page with a fixed size

# Filter by word count
ffuf -u http://TARGET/FUZZ -w wordlist.txt -fw 42
# Hide responses with exactly 42 words (common for "not found" pages)

# Filter by line count
ffuf -u http://TARGET/FUZZ -w wordlist.txt -fl 10
```

**How to find the right filter value:**
1. Run ffuf without filters first
2. Note the size/word count of the "not found" responses (they're all the same)
3. Re-run with `-fs SIZE` to hide those

```bash
# Example workflow
ffuf -u http://TARGET/FUZZ -w wordlist.txt
# Output shows: hundreds of results, all with size 1234
# Those are all "not found" pages — filter them out:
ffuf -u http://TARGET/FUZZ -w wordlist.txt -fs 1234
# Now only real results show up
```

---

## Subdomain/Virtual Host Discovery

```bash
ffuf -u http://TARGET -H "Host: FUZZ.target.com" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 1234
```

**What this does:** Instead of fuzzing the URL path, it fuzzes the `Host` header. This discovers virtual hosts — different websites hosted on the same server.

```
-H "Host: FUZZ.target.com"    Sets the Host header with FUZZ as subdomain
-fs 1234                       Filters out the default page (same size = not a real vhost)
```

**When you find a virtual host:** Add it to `/etc/hosts`:

```bash
echo "TARGET_IP dev.target.com" | sudo tee -a /etc/hosts
# Now browse to http://dev.target.com
```

---

## Parameter Fuzzing

### GET parameter discovery

```bash
ffuf -u "http://TARGET/page.php?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 1234
```

**What this does:** Tries different parameter names to find hidden parameters that the page accepts. If the page responds differently when you send `?debug=test` vs `?xyz=test`, then `debug` is a valid parameter.

### GET parameter value fuzzing

```bash
ffuf -u "http://TARGET/page.php?id=FUZZ" -w /usr/share/seclists/Fuzzing/1-1000.txt -fs 1234
```

**What this does:** Tries different values for a known parameter. Useful for IDOR testing — try IDs 1 through 1000 and see which ones return data.

### POST parameter fuzzing

```bash
ffuf -u http://TARGET/login.php -X POST -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -w /usr/share/wordlists/rockyou.txt -fc 401 -fs 1234
```

**Breaking this down:**

```
-X POST                         Use POST method (not GET)
-d "username=admin&password=FUZZ"  POST body with FUZZ in the password field
-H "Content-Type: ..."          Required header for POST form data
-w rockyou.txt                  Password wordlist
-fc 401                         Filter out 401 Unauthorized (failed logins)
-fs 1234                        Filter by size (if failed logins have a fixed size)
```

This is effectively brute forcing a login form — similar to Hydra but using ffuf.

---

## Multiple Wordlists (multi-position fuzzing)

```bash
ffuf -u "http://TARGET/FUZZ1/FUZZ2" -w wordlist1.txt:FUZZ1 -w wordlist2.txt:FUZZ2
```

**What this does:** Uses two wordlists simultaneously. Every combination of FUZZ1 and FUZZ2 is tried:

```
FUZZ1=admin, FUZZ2=login.php    → http://TARGET/admin/login.php
FUZZ1=admin, FUZZ2=config.php   → http://TARGET/admin/config.php
FUZZ1=portal, FUZZ2=login.php   → http://TARGET/portal/login.php
```

**Credential brute forcing with two wordlists:**

```bash
ffuf -u http://TARGET/login.php -X POST \
  -d "username=FUZZ1&password=FUZZ2" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -w users.txt:FUZZ1 -w passwords.txt:FUZZ2 \
  -fc 401
```

---

## Recursive Scanning

```bash
ffuf -u http://TARGET/FUZZ -w wordlist.txt -recursion -recursion-depth 2 -e .php -fc 404
```

**What `-recursion` does:** When ffuf finds a directory (301), it automatically scans inside that directory too. Just like feroxbuster.

```
Found: /admin/ (301)
  → Automatically scans: /admin/FUZZ
  → Found: /admin/panel/ (301)
    → Automatically scans: /admin/panel/FUZZ (depth 2)
```

`-recursion-depth 2` limits how deep it goes (default is unlimited — can be slow).

---

## Output Options

```bash
# Save results to a file
ffuf -u http://TARGET/FUZZ -w wordlist.txt -o results.json
ffuf -u http://TARGET/FUZZ -w wordlist.txt -o results.csv -of csv
ffuf -u http://TARGET/FUZZ -w wordlist.txt -o results.html -of html

# Quiet mode (less noise)
ffuf -u http://TARGET/FUZZ -w wordlist.txt -s
# Only shows found paths, no banner or progress info
```

---

## ffuf Flags Reference

| Flag | What it does | Example |
|---|---|---|
| `-u URL` | Target URL with FUZZ keyword | `-u http://TARGET/FUZZ` |
| `-w wordlist` | Wordlist file | `-w common.txt` |
| `-w file:NAME` | Named wordlist for multi-position | `-w users.txt:USER` |
| `-e .php,.txt` | Extensions to append | `-e .php,.txt,.bak` |
| `-X POST` | HTTP method | `-X POST` |
| `-d "data"` | POST data | `-d "user=FUZZ"` |
| `-H "Header"` | Custom header | `-H "Host: FUZZ.target.com"` |
| `-b "cookie"` | Cookie | `-b "session=abc123"` |
| `-mc 200,301` | Match status codes (show only these) | `-mc 200,301,403` |
| `-fc 404` | Filter status codes (hide these) | `-fc 404,500` |
| `-fs 1234` | Filter by response size | `-fs 0` (hide empty responses) |
| `-fw 42` | Filter by word count | `-fw 100` |
| `-fl 10` | Filter by line count | `-fl 5` |
| `-recursion` | Enable recursive scanning | |
| `-recursion-depth N` | Max recursion depth | `-recursion-depth 2` |
| `-t 50` | Number of threads | `-t 100` |
| `-rate 100` | Requests per second limit | `-rate 50` |
| `-o file` | Output file | `-o results.json` |
| `-of format` | Output format | `-of csv` |
| `-s` | Silent/quiet mode | |
| `-ic` | Ignore comments in wordlist | |
| `-v` | Verbose output | |

---

## ffuf vs gobuster — When to Use Each

```
USE FFUF WHEN:
  - Fuzzing parameters (GET/POST values)
  - Fuzzing headers (virtual host discovery with Host: header)
  - You need size/word/line filtering
  - Multi-position fuzzing (two wordlists)
  - You want recursive scanning with filtering
  - On the CPTS exam (it's the path tool)

USE GOBUSTER WHEN:
  - Quick directory brute force
  - DNS subdomain brute force (gobuster dns)
  - Simple, fast, fewer options to configure
  - On the OSCP exam (it's the standard tool)
```

For the CPTS, default to ffuf. It handles every web fuzzing scenario you'll face.
