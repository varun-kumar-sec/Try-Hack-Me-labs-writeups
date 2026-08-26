# Walkthrough : TryHackMe - Sudo Buffer Overflow
## Sudo Buffer Overflow Vulnerability Explanation
The Sudo Buffer Overflow (CVE-2019-18634) occurs in Sudo versions prior to 1.8.31 when the ```pwfeedback``` option is enabled in the ```/etc/sudoers``` file.
- Mechanism: When pwfeedback is enabled, Sudo provides visual feedback (such as displaying asterisks ```*```) on the terminal screen whenever a user types a password character.
- Flaw: The handling of user input during password prompt submission contains a flaw in how character erasure (such as pressing backspace or ```Ctrl+U```) is managed. Sending a large string of inputs combined with terminal control characters causes a buffer overflow in Sudo's internal line buffer.
- Exploitation: An unprivileged user can overwrite adjacent stack/heap data and manipulate execution flow, hijacking the privileged Sudo process to execute arbitrary commands directly as ```root``` without requiring a valid password.

---

## Step 1: Network Reconnaissance
- An Nmap scan is initiated against the target IP (```10.48.175.18```) to discover active ports and running services.
```bash
nmap -sV -sC -p- 10.48.175.18
```
- Results:
  - Port 2222/tcp: OpenSSH 7.6p1
  - Port 4444/tcp: OpenSSH 7.6p1

## Step 2: System Access & Environment Inspection
- Connect to the target machine via SSH on port ```4444``` using the local credentials (```tryhackme```).
```bash
ssh -p 4444 tryhackme@10.48.175.18
```
- Confirm user identity and inspect the home directory:
```bash
whoami
# Output: tryhackme

ls -la
```
- Findings:
  - A compiled binary named ```exploit``` is pre-staged in the home directory (```/home/tryhackme```).
  - Inspecting the binary using ```cat exploit``` confirms it targets the ```pwfeedback Sudo``` vulnerability (```SUDO_ASKPASS``` environment setup and execution logic).

## Step 3: Exploitation & Privilege Escalation
- Run the local exploit binary to trigger the buffer overflow condition within Sudo:
```bash
./exploit
```
- Execution Flow:
  - The exploit feeds crafted input into the Sudo prompt, causing the buffer to overflow and overriding authentication flow control.
  - Even though the terminal outputs ```Sorry, try again```., the payload successfully spawns an elevated shell process.

- Verify the spawned shell's context:
```bash
whoami
# Output: root
```
## Step 4: Flag Extraction
- Access the /root directory and display the final challenge flag:
```bash
cd /root
ls
cat root.txt
```
- Root Flag: ```THM{buff3r_0v3rfl0w_rul3s}```
