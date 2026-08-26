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
