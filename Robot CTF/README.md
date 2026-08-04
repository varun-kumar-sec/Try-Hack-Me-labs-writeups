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

---

## Next Steps

With credentials for the `elliot` user account now recovered, the next logical step in the assessment is authenticating via the WordPress login interface at `/wp-login.php` to achieve administrative access within the CMS.
