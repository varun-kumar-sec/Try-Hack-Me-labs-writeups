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
    
    Access http://<TARGET-IP>/ in a browser and inspect the HTML page source.
    Search for hidden comments, subdirectories, or potential usernames (e.g., boris, natalya, knut).
    Brute-force POP3 logins using discovered usernames against port 55007.

### 3. POP3 Brute-Forcing (boris / natalya)

Use Hydra or Medusa to crack credentials for POP3 users:
```hydra -l natalya -P /usr/share/wordlists/rockyou.txt <TARGET-IP> -s 55007 pop3```

## Moodle Access & Secondary POP3 Enumeration

### 4. POP3 Mail Enumeration (natalya)

Connect via POP3 to inspect Natalya's emails:

    Message 1: Mentions supervisor details and warns about the crime syndicate Janus.

    Message 2: Contains credentials for xenia and internal routing instructions:

        Username: xenia

        Password: RCP90rulez!

        Internal Domain Endpoint: severnaya-station.com/gnocertdir

### 5. Local DNS Resolution (/etc/hosts)

Add the local domain entry to /etc/hosts so your browser can resolve the internal path
```sudo vim /etc/hosts```

Append entry:
```<TARGET-IP> severnaya-station.com```

### 6. Moodle Portal Access & Secondary POP3 Crack

    Navigate to http://severnaya-station.com/gnocertdir/ in your browser.
    Log in with Xenia's credentials (xenia / RCP90rulez!).
    Inspecting internal messages/profile reveals contact with Dr Doak.
    Use Hydra to brute-force POP3 credentials for user doak:

    hydra -l doak -P /usr/share/set/src/fasttrack/wordlist.txt <TARGET-IP> -s 55007 pop3
Discovered POP3 Credentials (doak):

    Username: doak

    Password: goat

### 7. Moodle Login (dr_doak) & Private Files Exploration

    Log into POP3 (doak / goat) via telnet or nc to inspect email contents:

        Retrieved Web Credentials:

            Username: dr_doak

            Password: 4England!

            Log into the Moodle platform as dr_doak.

            Access My profile $\rightarrow$ My private files to find:
            
            Folder: for james
            
            File: s3cret.txt
