# Walkthrough : TryHackMe - Pickle Rick
## Executive Summary
This report documents the end-to-end exploitation of the Pickle Rick CTF lab. Reconnaissance revealed an active web server hosting a customized portal. Source code inspection and configuration files leaked valid admin credentials, granting access to a web-based command panel. Full system root privilege was achieved via misconfigured sudo permissions without needing a full reverse shell connection.

## 1. Reconnaissance & Initial Access
- Network Scanning: An Nmap scan of target IP ```10.49.167.107``` identified two active services:
  - ```22/tcp``` — OpenSSH 8.2p1 (Ubuntu)
  - ```80/tcp``` — Apache httpd 2.4.41 (Ubuntu)

- Web Enumeration:
  - Inspecting the source code of ```[http://10.49.167.107/](http://10.49.167.107/)``` revealed an embedded comment exposing the username: ```R1ckRul3s```.
  - Retrieving ```[http://10.49.167.107/robots.txt](http://10.49.167.107/robots.txt)``` exposed the plaintext string: ```Wubbalubbadubdub```.

- Portal Authentication:
  - Navigated to ```/login.php``` and successfully logged in using ```R1ckRul3s : Wubbalubbadubdub```, landing on the Command Panel (```/portal.php```).

## 2. Exploitation & Privilege Escalation
- Command Execution (Web Shell):
  - Executed ```ls -la /var/www/html``` to list the web root directory contents.
  - Located ```Sup3rS3cretPickl3Ingred.txt``` containing the first ingredient.
- System Navigation:
  - Executed ```ls -la /home``` and identified user directory ```/home/rick```.
  - Located ```/home/rick/second``` ingredients containing the second ingredient.

- Root Escalation & Final Flag Recovery:
  - Executed ```sudo -l``` within the portal command input, revealing full passwordless Sudo privileges for ```www-data```:
```bash
User www-data may run the following commands on ip-10-49-167-107:
    (ALL) NOPASSWD: ALL
```
- Leveraged Sudo to bypass standard command restrictions and read ```/root/root.txt```:
```bash
sudo less '/root/root.txt'
```
- Retrieved the third ingredient.

## 3. Summary of Recovered Ingredients (Flags)

| Flag Description  | Source Path / Vector                        | Retrieved Value   |
|:--                |:--                                          |:--                |
| 1st Ingredient    | /var/www/html/Sup3rS3cretPickl3Ingred.txt   | mr. meeseek hair  | 
| 2nd Ingredient    | /home/rick/second ingredients               | 1 jerry tear      | 
| 3rd Ingredient    | /root/root.txt (via sudo less)              | fleeb juice       | 
