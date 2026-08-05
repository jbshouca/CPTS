# 02 — Network Enumeration with Nmap

Nmap is your first tool on every engagement. On the CPTS, thorough scanning is even more critical than on the OSCP because the enterprise network has multiple subnets, hidden services, and hosts you can only discover through pivoting. Missing a port means missing an attack vector.

---

## Scan Types Explained

### Host discovery (is anything alive?)

```bash
nmap -sn 10.10.10.0/24
```

**What `-sn` does:** Ping scan — sends ICMP echo, TCP SYN to port 443, TCP ACK to port 80, and an ICMP timestamp request. If any of these get a response, the host is marked as "up." No port scanning is performed.

**Why you need this:** Before scanning ports, know which IPs are alive. Scanning 254 IPs × 65,535 ports each wastes hours. Scanning only the 12 alive hosts is fast.

```bash
# If ICMP is blocked (common on hardened networks)
nmap -sn -PS22,80,443,445 10.10.10.0/24
# -PS = TCP SYN ping on specific ports (more likely to get through firewalls)

# ARP scan (fastest on local subnets — layer 2, can't be firewalled)
nmap -sn -PR 10.10.10.0/24

# Save alive hosts for later
nmap -sn 10.10.10.0/24 -oG - | grep "Up" | awk '{print $2}' > alive.txt
```

### TCP SYN scan (the default — fast and stealthy)

```bash
sudo nmap -sS 10.10.10.15
```

**What `-sS` does:** Sends a SYN packet to each port. If the port responds with SYN-ACK, it's open. Nmap then sends RST (reset) to close the connection WITHOUT completing the three-way handshake. This is why it's called "half-open" scanning — the connection is never fully established.

```
Kali → SYN → Target port 80
Target port 80 → SYN-ACK → Kali    (port is OPEN)
Kali → RST → Target                 (close without connecting)

Kali → SYN → Target port 81
Target port 81 → RST → Kali         (port is CLOSED)

Kali → SYN → Target port 82
... no response ...                  (port is FILTERED — firewall dropped it)
```

**Requires root** because crafting raw SYN packets needs raw socket access.

### TCP connect scan (when you don't have root)

```bash
nmap -sT 10.10.10.15
```

**What `-sT` does:** Completes the full TCP three-way handshake (SYN → SYN-ACK → ACK). Slower and noisier than SYN scan, but doesn't require root privileges. **Use this through proxychains** — SYN scan doesn't work through SOCKS proxies because raw packets can't be proxied.

```bash
# Through a tunnel (MUST use -sT)
proxychains nmap -sT -Pn 172.16.0.10
```

### UDP scan (don't skip this)

```bash
sudo nmap -sU --top-ports 50 10.10.10.15
```

**What `-sU` does:** Sends UDP packets to each port. UDP is connectionless — there's no SYN-ACK to confirm a port is open. If the port responds, it's open. If it responds with ICMP "port unreachable," it's closed. If there's no response... it could be open OR filtered. That's why UDP scanning is slow and uncertain.

**Why you can't skip it:**
- DNS (53/udp) — zone transfers, subdomain info
- SNMP (161/udp) — dumps massive amounts of system info
- TFTP (69/udp) — unauthenticated file access
- NTP (123/udp) — sometimes reveals internal hostnames
- DHCP (67-68/udp) — network configuration info

```bash
# SNMP specifically (the most common UDP finding on pentests)
sudo nmap -sU -p 161 --script snmp-info 10.10.10.0/24
```

---

## Service and Version Detection

```bash
nmap -sV 10.10.10.15
```

**What `-sV` does:** After finding open ports, nmap sends protocol-specific probes to identify the service and its version. For HTTP, it sends an HTTP request. For SSH, it reads the banner. For SMB, it negotiates the protocol.

```
Without -sV:  22/tcp open ssh
With -sV:     22/tcp open ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.6

The version string "OpenSSH 8.9p1 Ubuntu 3ubuntu0.6" tells you:
  - Software: OpenSSH 8.9p1
  - OS: Ubuntu
  - searchsploit openssh 8.9 → check for CVEs
```

**Version detection intensity:**

```bash
nmap -sV --version-intensity 5 10.10.10.15    # more probes, slower, more accurate
# Default intensity is 7 (0-9 scale, higher = more probes)
```

---

## NSE Scripts (Nmap Scripting Engine)

```bash
nmap -sC 10.10.10.15
```

**What `-sC` does:** Runs the "default" category of NSE scripts. These perform additional enumeration — reading SSL certificates, checking for anonymous FTP, listing SMB shares, checking for known vulnerabilities.

```bash
# Common combinations
nmap -sV -sC 10.10.10.15              # version + default scripts
nmap -sV -sC -p- 10.10.10.15          # all ports + version + scripts

# Specific scripts
nmap --script smb-enum-shares -p 445 10.10.10.15
nmap --script http-shellshock --script-args uri=/cgi-bin/status.sh -p 80 10.10.10.15
nmap --script vuln -p 445 10.10.10.15
nmap --script ssl-enum-ciphers -p 443 10.10.10.15

# Script categories
nmap --script=vuln 10.10.10.15         # check for known vulnerabilities
nmap --script=auth 10.10.10.15         # check for auth issues (default creds)
nmap --script=safe 10.10.10.15         # non-intrusive scripts only
```

---

## The CPTS Scanning Workflow

### Phase 1: Quick scan (first 5 minutes)

```bash
nmap -sV -sC -oA quick 10.10.10.0/24
```

Get results fast. Start working with what comes back while the full scan runs.

### Phase 2: Full port scan (background)

```bash
nmap -p- --min-rate=1000 -oA full 10.10.10.0/24 &
```

```
-p-              Scan ALL 65,535 ports (not just the top 1000)
--min-rate=1000  Send at least 1000 packets per second (faster)
&                Run in background
```

**Why full port scans matter:** The default nmap scan checks only the top 1000 ports. Applications often run on non-standard ports (8080, 8443, 8888, 9090, 3000, 5000). You'll miss Jenkins on 8080 or Grafana on 3000 if you don't do a full scan.

### Phase 3: Targeted deep scan (after full scan completes)

```bash
# Check full scan results for new ports
diff <(grep "open" quick.nmap) <(grep "open" full.nmap)

# Deep scan any new ports found
nmap -sV -sC -p 8080,8443,9090 10.10.10.15
```

### Phase 4: UDP scan (don't forget)

```bash
sudo nmap -sU --top-ports 50 -oA udp 10.10.10.0/24
```

### Phase 5: Internal network scan (after pivoting)

```bash
# Through a SOCKS proxy (MUST use -sT, no -sS)
proxychains nmap -sT -Pn -sV -p 22,80,443,445,3389,5985 172.16.0.0/24

# -Pn = skip host discovery (can't ping through proxychains reliably)
# -sT = TCP connect (required through proxychains)
```

---

## Output Formats

```bash
nmap -sV -sC -oA scan_name TARGET
```

**What `-oA` does:** Saves results in ALL three formats simultaneously:

```
scan_name.nmap    Normal text output (human readable)
scan_name.gnmap   Greppable output (one host per line — easy to parse)
scan_name.xml     XML output (for importing into other tools)
```

```bash
# Other output options
-oN file.nmap     Normal output only
-oG file.gnmap    Greppable output only
-oX file.xml      XML output only
```

### Parsing greppable output

```bash
# Find all hosts with port 80 open
grep "80/open" scan.gnmap | awk '{print $2}'

# Find all hosts with SMB
grep "445/open" scan.gnmap | awk '{print $2}'

# Find all hosts with SSH
grep "22/open" scan.gnmap | awk '{print $2}' > ssh_hosts.txt
```

---

## Nmap Flags Reference

| Flag | What it does | When to use |
|---|---|---|
| `-sn` | Ping scan (host discovery only) | First step — find alive hosts |
| `-sS` | SYN scan (half-open, stealthy) | Default scan, needs root |
| `-sT` | TCP connect scan (full handshake) | Through proxychains, without root |
| `-sU` | UDP scan | Finding SNMP, DNS, TFTP |
| `-sV` | Version detection | Always — identifies software versions |
| `-sC` | Default scripts | Always — extra enumeration |
| `-p-` | All 65,535 ports | Full scan — find hidden services |
| `-p 22,80,443` | Specific ports | Targeted scan after discovery |
| `--min-rate=1000` | Minimum packet rate | Speed up full port scans |
| `-Pn` | Skip host discovery | When ICMP is blocked, through proxies |
| `-oA name` | All output formats | Always save your scans |
| `--script=name` | Specific NSE script | Targeted vulnerability checks |
| `-A` | Aggressive (OS detect + version + scripts + traceroute) | Quick comprehensive scan |
| `-T4` | Timing template (faster) | Default for most scans |
| `-v` | Verbose output | See results as they come in |
