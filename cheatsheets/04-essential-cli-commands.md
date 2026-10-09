# Essential CLI Commands Cheatsheet for Technical Rounds

> **Cybersecurity Interview Focus:** Rapid recall of terminal commands across Linux, Windows PowerShell, Nmap, OpenSSL, Wireshark/tcpdump, Git, and Docker.

---

## 1. Linux & Bash Networking & Process Investigation

```bash
# Process Investigation
ps aux | grep <process_name>          # List running processes with user, PID, CPU%
top -b -n 1                           # Snapshot of resource-hungry processes
kill -9 <PID>                         # Force kill stubborn process
lsof -i :<port>                       # Identify process listening on specific port
lsof -p <PID>                         # List all open files and sockets held by PID

# Network Sockets & Connections
ss -tulnp                             # Modern alternative to netstat (listening TCP/UDP + PIDs)
netstat -antp                         # Classic socket table with connection states
ip a / ifconfig                       # Interface IP addresses & MAC addresses
ip route show                         # Show kernel IP routing table

# File Permissions & Ownership
chmod 750 <file>                      # rwxr-x--- (User: all, Group: read/exec, Other: none)
chmod u+s /path/binary                # Set SUID bit (runs with owner privileges)
chown root:security <file>            # Change owner to root and group to security
find / -perm -u=s -type f 2>/dev/null # Hunt for SUID binaries (classic Linux PrivEsc check)

# Text Processing & Log Parsing
grep -rn "Failed password" /var/log/auth.log     # Search recursively with line numbers
awk '{print $1}' access.log | sort | uniq -c     # Count unique visitor IPs from web log
sed -i 's/PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config # In-place config edit
```

---

## 2. Windows & PowerShell Security Investigation

```powershell
# Sockets and Network
Get-NetTCPConnection -State Listen | Select-Object LocalPort, OwningProcess
Test-NetConnection -ComputerName target.com -Port 443   # PowerShell ping + port check (TCP 3-way)

# Processes and Services
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Service | Where-Object {$_.Status -eq "Running"}
Get-CimInstance Win32_Service | Select-Object Name, StartName, PathName # Unquoted service path check

# Windows Event Log Triage
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 20 # Failed Logins (Event 4625)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 20 # Successful Logins (Event 4624)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4720}                # User Account Created
```

---

## 3. Nmap Scanning Commands

```bash
nmap -sS -p- -T4 <target_ip>          # SYN Stealth scan across all 65535 ports
nmap -sV -sC -p 22,80,443 <target_ip> # Version detection (-sV) + Default security scripts (-sC)
nmap -O <target_ip>                   # OS fingerprinting
nmap --script vuln <target_ip>        # Automated vulnerability scanning scripts
nmap -sn 192.168.1.0/24               # Ping sweep / host discovery without port scanning
```

---

## 4. Packet Capture & Analysis (tcpdump & TShark)

```bash
tcpdump -i eth0 -nn -c 100                     # Capture 100 packets without resolving hostnames/ports
tcpdump -i eth0 'tcp port 80 and host 10.0.0.5' # Filter for HTTP traffic to/from specific host
tcpdump -i any 'tcp[tcpflags] & (tcp-syn) != 0 and tcp[tcpflags] & (tcp-ack) == 0' # Isolate SYN packets
tcpdump -w capture.pcap -i eth0                # Write raw packets to file for Wireshark inspection
```

---

## 5. OpenSSL Diagnostic & Certificate Verification

```bash
# Test TLS Handshake and Inspect Certificate Chain
openssl s_client -connect example.com:443 -tls1_3

# Check Certificate Expiration and Subject
openssl x509 -in cert.pem -text -noout | grep -E "Not After|Subject:"

# Calculate Cryptographic Hashes
openssl dgst -sha256 file.zip
sha256sum file.zip
```

---

## 6. Docker & Container Security Commands

```bash
docker ps -a                          # List all containers (running & stopped)
docker exec -it <container_id> sh     # Open shell inside running container
docker logs --tail 100 <container_id> # Inspect recent container stdout/stderr logs
docker inspect --format='{{.HostConfig.Privileged}}' <container> # Check if running privileged (RISK!)
docker scout quickview / trivy image <image_name>               # Scan container image for CVEs
```
