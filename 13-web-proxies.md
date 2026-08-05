# 13 — Using Web Proxies (Burp Suite)

Burp Suite is your primary tool for web application testing on the CPTS. The HTB Academy module covers it in depth because the exam requires manual web testing — not just running automated scanners. You need to intercept requests, modify them, replay them, and fuzz parameters through Burp.

---

## Burp Suite Architecture

```
Your Browser → Burp Proxy (127.0.0.1:8080) → Target Web Server
                    │
                    ├── HTTP History (records every request)
                    ├── Interceptor (pause and modify requests)
                    ├── Repeater (replay and modify manually)
                    ├── Intruder (automated fuzzing/brute forcing)
                    ├── Decoder (encode/decode data)
                    └── Comparer (diff two responses)
```

**Everything passes through Burp.** That means you see every request your browser sends and every response the server returns — including hidden headers, cookies, redirects, and API calls that the browser handles silently.

---

## Setup for the CPTS Exam

### Configure Firefox

```
Firefox → Settings → search "proxy" → Manual proxy configuration
  HTTP Proxy: 127.0.0.1    Port: 8080
  Check: Also use this proxy for HTTPS

Install Burp's CA certificate:
  Browse to http://burp → download cacert.der
  Firefox → Settings → Privacy → View Certificates → Import → cacert.der
  Check both trust boxes
```

### Configure Burp for efficiency

```
Project options → HTTP → Redirections → check "Follow redirections"
  (so you see the final page, not just the 302)

Proxy → Options → Intercept Client Requests:
  Uncheck "Intercept requests based on..." initially
  Only turn on intercept when you specifically need to modify a request

Target → Scope:
  Add your target URL to scope
  Then Proxy → Options → "Only show in-scope items" in HTTP history
  This filters out noise from other sites
```

---

## Proxy — Recording and Intercepting

### HTTP History (passive recording)

With intercept OFF, every request flows through Burp and gets recorded. This is your primary intelligence source:

```
Browse the entire web application normally with intercept OFF.
Every request appears in Proxy → HTTP History.

What to look for in the history:
  - Every URL path the application uses
  - Parameters in GET requests (?id=1&action=view)
  - POST bodies (login forms, search forms, API calls)
  - Cookies being set and their values
  - API endpoints called by JavaScript (AJAX requests)
  - Hidden redirects
  - Error responses (500s might reveal stack traces)
```

**Click any request in the history** to see the full request and response side by side. This is where you identify injection points.

### Intercepting requests (active modification)

Turn intercept ON when you want to modify a request before it reaches the server:

```
Use case 1: Modify a hidden form field
  Form has: <input type="hidden" name="role" value="user">
  Intercept the POST, change role=user to role=admin, Forward

Use case 2: Bypass client-side validation
  JavaScript prevents special characters in the search box
  Intercept the request, add SQLi payload, Forward
  Client-side validation is bypassed — the server receives your payload

Use case 3: Modify cookies
  Cookie: session=abc123; user_level=1
  Intercept, change user_level=1 to user_level=9, Forward
  If the server trusts the cookie value → privilege escalation
```

**Intercept workflow:**

```
1. Turn Intercept ON (Proxy → Intercept → "Intercept is on")
2. Perform an action in the browser (click a button, submit a form)
3. The request appears in Burp (the browser waits)
4. Modify any part of the request
5. Click Forward (sends the modified request to the server)
6. Or click Drop (cancels the request entirely)
7. Turn Intercept OFF when done (so normal browsing resumes)
```

---

## Repeater — Manual Testing Workhorse

Repeater is where you spend most of your time on the CPTS. Send a request, modify it, send again, compare responses.

### Workflow

```
1. In HTTP History, right-click a request → Send to Repeater (Ctrl+R)
2. Switch to the Repeater tab
3. The request is on the left, response on the right
4. Modify the request → click Send → see the new response
5. Modify again → Send again → compare
```

### SQLi testing in Repeater

```
Request tab shows:
  GET /products.php?cat=Electronics HTTP/1.1
  Host: target.com
  ...

Test 1: Change cat=Electronics to cat=Electronics'
  → Send → check response for SQL error

Test 2: Change to cat=' UNION SELECT 1,2,3-- -
  → Send → check if numbers appear in the response

Test 3: Change to cat=' UNION SELECT 1,@@version,3-- -
  → Send → database version appears

You iterate quickly without re-clicking through the browser each time.
```

### IDOR testing in Repeater

```
Original request:
  GET /api/user/profile?id=123 HTTP/1.1
  Cookie: session=YOUR_SESSION

Change id=123 to id=124 → Send
  → If you see user 124's data → IDOR confirmed

Change id=124 to id=125 → Send
  → Continue enumeration

Change id=1 → Send
  → Check if user 1 is an admin with different data
```

### XXE testing in Repeater

```
Original request:
  POST /api/submit HTTP/1.1
  Content-Type: application/xml
  
  <data><name>test</name></data>

Modify the body to:
  <?xml version="1.0"?>
  <!DOCTYPE data [
    <!ENTITY xxe SYSTEM "file:///etc/passwd">
  ]>
  <data><name>&xxe;</name></data>

Send → check if /etc/passwd content appears in the response
```

### Response comparison

```
Use case: Boolean-based blind SQLi

Request 1: ?id=1 AND 1=1    → Response length: 4523 (normal content)
Request 2: ?id=1 AND 1=2    → Response length: 1205 (no content)

The length difference confirms blind SQLi.
You can't see this in the browser — Repeater shows exact lengths.
```

---

## Intruder — Automated Attacks

Intruder sends many requests with different payloads. Used for brute forcing, parameter fuzzing, and IDOR enumeration.

### Attack types

```
Sniper:      One position, one wordlist.
             Each word replaces the position one at a time.
             Use for: single parameter testing, brute forcing

Battering Ram: One wordlist, ALL positions get the SAME value each time.
               Use for: testing the same value in multiple places

Pitchfork:   Multiple positions, multiple wordlists.
             Position 1 gets word 1 from list 1, position 2 gets word 1 from list 2.
             Use for: known username:password pairs

Cluster Bomb: Multiple positions, ALL COMBINATIONS of all wordlists.
              Use for: brute forcing login (all usernames × all passwords)
```

### IDOR enumeration with Intruder

```
1. Send the request to Intruder (Ctrl+I)
2. Clear all positions → highlight just the ID value → Add §
   GET /api/user/§123§ HTTP/1.1
3. Payloads tab → Payload type: Numbers
   From: 1, To: 1000, Step: 1
4. Start attack
5. Sort results by response length — different lengths = valid users
```

### Login brute force with Intruder

```
1. Send the login POST to Intruder
2. Attack type: Cluster Bomb
3. Positions:
   POST /login HTTP/1.1
   username=§admin§&password=§password§
4. Payload set 1: username wordlist
   Payload set 2: password wordlist
5. Start attack
6. Filter results: look for different status code or response length
```

**Note:** Burp Community Edition throttles Intruder to 1 request/second. For real brute forcing, use ffuf or Hydra. Intruder is best for small targeted attacks (100-1000 attempts).

---

## Decoder — Encode/Decode Data

Use Decoder to transform data you find in requests and responses:

```
Common transformations:
  Base64 decode: eyJ1c2VyIjoiYWRtaW4ifQ== → {"user":"admin"}
  URL decode:    %3Cscript%3E → <script>
  HTML decode:   &lt;script&gt; → <script>
  Hex decode:    48656c6c6f → Hello

Use case: You find a cookie value that looks like base64
  Cookie: data=eyJ1c2VyIjoiam9obiIsInJvbGUiOiJ1c2VyIn0=
  Decode: {"user":"john","role":"user"}
  
  Modify: {"user":"john","role":"admin"}
  Encode back to base64: eyJ1c2VyIjoiam9obiIsInJvbGUiOiJhZG1pbiJ9
  
  Set the new cookie → admin access
```

---

## Saving Requests for SQLMap

One of the most common CPTS workflows — capture in Burp, exploit with SQLMap:

```
1. In HTTP History, find the request with the injectable parameter
2. Right-click → Save item
3. Save as: request.txt
4. On the command line:
   sqlmap -r request.txt --batch --dbs
```

**Why this is better than building the URL manually:**
- Captures all cookies (authentication cookies)
- Captures all headers (CSRF tokens, custom headers)
- Captures the exact content type
- Captures POST body formatting
- No encoding issues

---

## Burp for the CPTS Exam — Key Workflows

```
WORKFLOW 1: Web app reconnaissance
  1. Add target to scope
  2. Browse the entire app with intercept OFF
  3. Review HTTP history — note every endpoint and parameter
  4. Identify technology from headers and cookies
  5. Note any API calls made by JavaScript

WORKFLOW 2: Testing for injection
  1. Send interesting requests to Repeater
  2. Test each parameter: add ', ", ;, |, <, {{, etc.
  3. Compare response lengths and error messages
  4. When injection confirmed → exploit in Repeater or hand off to SQLMap

WORKFLOW 3: IDOR testing
  1. Identify requests with object references (IDs, filenames)
  2. Send to Repeater → change the ID → observe response
  3. For scale, send to Intruder → enumerate all valid IDs

WORKFLOW 4: Authentication bypass
  1. Intercept login request
  2. Modify hidden fields, cookies, or headers
  3. Try SQLi in username/password fields
  4. Try verb tampering (change POST to PUT)
  5. Try adding admin parameters (?admin=true, role=admin)

WORKFLOW 5: Save request → SQLMap
  1. Identify injectable parameter in Repeater
  2. Save item → request.txt
  3. sqlmap -r request.txt --batch --dbs
```

---

## Keyboard Shortcuts

```
Ctrl+R    Send to Repeater
Ctrl+I    Send to Intruder
Ctrl+U    URL-encode selected text
Ctrl+Shift+U    URL-decode selected text
Ctrl+B    Base64-encode selected text
Ctrl+Shift+B    Base64-decode selected text
Tab       Switch between request and response panels
```
