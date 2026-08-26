# Walkthrough : TryHackMe - Linux Agency
## Executive Summary
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
