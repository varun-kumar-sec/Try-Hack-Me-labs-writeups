# TryHackMe: Juicy Details — SOC Log Analysis & Walkthrough
Room: Juicy Details
Role: SOC Analyst
Category: Digital Forensics & Incident Response (DFIR) / Log Analysis
Tools Used: ```grep```, ```cat```, Linux CLI utilities

## 1. Incident Overview
As a Security Operations Center (SOC) Analyst investigating a breach on the "Juice Shop" environment, server log files (```access.log```, ```auth.log```, ```vsftpd.log```) were analyzed to reconstruct the adversary's attack lifecycle. The investigation identified the attacker's automated scanning tools, exploited web endpoints, exfiltrated sensitive data, and system service attack vectors.

## 2. Reconnaissance & Tool Identification
By inspecting incoming User-Agent strings and traffic patterns within ```access.log```, the threat actor's toolset was identified in chronological order of occurrence.

Attacker Tooling Breakdown
| Sequence        | Tool Name                                                      | User-Agent / Log Signature,Activity / Purpose                         |
|:--              |:--                                                             |:--                                                                    |
| 1,Nmap          | Mozilla/5.0 (compatible; Nmap Scripting Engine; ...)           | "Initial host discovery, port scanning, and NSE script enumeration."  |
| 2,Hydra         | Mozilla/5.0 (Hydra)                                            | High-speed HTTP POST login brute-force attack.                        | 
| 3,sqlmap        | sqlmap/1.5.2#stable ([http://sqlmap.org](http://sqlmap.org))   | Automated SQL injection testing and database schema dumping.          | 
| 4,cURL          | curl/7.74.0                                                    | Manual HTTP requests used to verify exfiltration endpoints.           | 
| 5,Feroxbuster   | feroxbuster/2.2.1                                              | Recursive directory and file path brute-forcing.                      |

Vulnerable Web Endpoints & Vectors
- Brute-Force Target: ```/rest/user/login```
  - Vector: Attacker sent rapid ```POST``` requests via Hydra attempting to guess credentials.
- SQL Injection Vulnerability: ```/rest/products/search```
  - Vector: Parameter ```q``` (```/rest/products/search?q=...```) was vulnerable to union-based SQL injection, allowing arbitrary database queries.
- File Directory Exposure Endpoint: ```/ftp```
  - Vector: Exposed via Feroxbuster, leading to unauthenticated directory listing and backup file access.

## 3. Threat Analysis & Stolen Data Verification
An in-depth log review revealed how the attacker gathered intelligence, authenticated, extracted database credentials, and retrieved sensitive files.

Investigation Findings Matrix
| Analysis Question                  | Log Evidence / Finding                                                                                         | Final Answer                         |
|:--                                 |:--                                                                                                             |:--                                   |
| Email Address Scraping Path        | "Repeated GET requests to review IDs (e.g., /rest/products/4/reviews) to harvest commenter emails."            | product reviews                      |
| Successful Brute-Force Timestamp   | Single POST /rest/user/login request returning HTTP 200 with payload size 831 (versus 401 size 26).            | "Yay, 11/Apr/2021:09:16:31 +0000"    | 
| Exfiltrated SQL Data               | Injected UNION SELECT statements explicitly targeted database columns: email and password.                     | "email, password"                    |
| Exfiltrated Backup Files           | GET requests sent to the exposed /ftp endpoint to download sensitive backup files.                             | "coupons_2013.md.bak, www-data.bak"  | 
| File Retrieval Service & User      | File transfer attempts were logged via FTP using public/anonymous login credentials.                           | "ftp, anonymous"                     |

## 4. Detailed Technical Explanations
Log Analysis Mechanics
- Identifying HTTP Brute-Force Success: When inspecting web server logs, failed authentication attempts typically result in identical HTTP ```401 Unauthorized``` responses with small byte sizes (e.g., ```26``` bytes). A successful attempt produces a distinct anomaly—in this case, an HTTP ```200``` OK with a response length of ```831``` bytes, signaling a session cookie or JWT token payload was issued.
- SQL Injection (SQLi) Infiltration: The ```sqlmap``` queries exploited the ```q``` parameter using ```UNION SELECT``` techniques (```UNION SELECT id, email, password FROM Users--```). This forced the database to return stored user table contents inside the product search results page.
- Sensitive File Exposure via ```/ftp```: Backup files ending in ```.bak``` located inside public directories represent significant risk. Here, ```coupons_2013.md.bak``` leaked business logic data, while ```www-data.bak``` exposed system configurations.
- Privilege Escalation & SSH Pivoting: Attackers often pivot from web app access to low-level system access. Logs in ```auth.log``` showed the actor attempting SSH login attempts against the ```www-data``` service account on various source ports.

## 5. Defense & Remediation Recommendations
1. Implement Rate Limiting & Account Lockouts: Limit repeated login attempts on ```/rest/user/login``` to mitigate automated credential stuffing tools like Hydra.
2. Use Parameterized Queries (Prepared Statements): Sanitize all input fields on ```/rest/products/search``` to neutralize SQL injection vulnerabilities.
3. Restrict Access to Sensitive File Paths: Remove ```.bak``` files from public web directories (```/ftp```) and disable directory listing.
4. Disable Shell Access for Service Accounts: Restrict login shells for system-level service accounts (such as ```www-data```) by setting their shell to ```/usr/sbin/nologin``` or ```/dev/null``` in ```/etc/passwd```.
