# Walkthrough : TryHackMe - Http in Detail
## Module 1: What is HTTP(S)?
1. What is HTTP? (HyperText Transfer Protocol)
- The Core Definition: HTTP is a set of rules (a protocol) used by devices to communicate across the internet.
- Primary Purpose: It is used whenever you load or view a website to transfer data—such as text (HTML), images, videos, and files—from a web server directly to your browser.
- History: Created by Tim Berners-Lee and his team between 1989–1991.

2. What is HTTPS? (HyperText Transfer Protocol Secure)
- The Core Definition: HTTPS is simply the secure, encrypted version of HTTP.
- Encryption (Privacy): It scrambles the data sent back and forth so that attackers, Wi-Fi eavesdroppers, or ISPs cannot read sensitive information like passwords or credit card numbers.
- Server Verification (Trust): It ensures you are connected to the actual, legitimate web server instead of a malicious server pretending to be it.

3. Key Takeaway & Comparison
- HTTP: Transmits data in plain text. Fast, but unsafe for sensitive information.
- HTTPS: Transmits data via an encrypted tunnel. Safe, private, and secure.

Lab Questions Reference
- Q: What does HTTP stand for?
  - A: ```HyperText Transfer Protocol```
- Q: What does HTTPS stand for?
  - A: ```HyperText Transfer Protocol Secure```
- Q: On the mock webpage on the right there is an issue, once you've found it, click on it. What is the challenge flag?  
  - A: ```THM{INVALID_HTTP_CERT}```

## Module 2: Requests and Responses
1. Anatomy of a URL (Uniform Resource Locator)
A URL is an instruction on how and where to access a resource on the internet.
- Scheme: Specifies the protocol used to access the resource (e.g., ```HTTP```,``` HTTPS```, ```FTP```).
- User: Credentials included directly in the URL for services requiring authentication.
- Host: The domain name or IP address of the target server.
- Port: The connection port on the server (default ```80``` for HTTP, ```443``` for HTTPS; range ```1-65535```).
- Path: The specific file name or directory path being accessed on the server.
- Query String: Parameters passed to the path using ```?key=value``` syntax (e.g., ```?id=1```).
- Fragment: A reference (```#```) pointing directly to a specific section on the requested page.

2. HTTP Requests
An HTTP request is sent by the browser to ask the server for data. Minimal requests can consist of just one request line.
- Structure Breakdown:
  - Line 1 (Request Line): Contains the request method (e.g., ```GET```), the path requested (```/```), and the protocol version (```HTTP/1.1```).
  - Host Header: Specifies the domain name of the web server (```tryhackme.com```).
  - User-Agent Header: Identifies the client's browser and operating system details (```Mozilla/5.0 Firefox/87.0```).
  - Referer Header: Indicates the web page that linked to the current request ```([https://tryhackme.com/](https://tryhackme.com/))```.
  - Blank Line: Every HTTP request ends with an empty line to signal the server that transmission is complete.

3. HTTP Responses
An HTTP response is sent back by the server containing status information and requested content.
- Structure Breakdown:
  - Line 1 (Status Line): Returns protocol version (```HTTP/1.1```) and the HTTP status code (```200 OK```) indicating success.
  - Server Header: Details the web server software and version running on the target (```nginx/1.15.8```).
  - Date Header: Timestamp and timezone of the server response.
  - Content-Type Header: Specifies the media format of the returned data (e.g., ```text/html```, images, XML).
  - Content-Length Header: Indicates total size of the response body in bytes to ensure no data loss.
  - Blank Line: Separates response headers from the response payload.
  - Body: The actual requested content payload (e.g., HTML structure).

Lab Questions Reference
- What HTTP protocol is being used in the above example?
```HTTP/1.1```
- What response header tells the browser how much data to expect?
```Content-Length```

## Module 3: HTTP Methods
1. Overview
HTTP methods tell the web server what action the client wants to perform. While there are several methods, ```GET``` and ```POST``` are the two most commonly used across the web.

2. Core HTTP Methods Breakdown
- GET Request: Used to retrieve or view information from a web server (e.g., loading a article or viewing a profile).
- POST Request: Used to submit new data to the server, often creating new records (e.g., registering an account or making a post).
- PUT Request: Used to send data to a web server to update existing information (e.g., changing your email or editing profile settings).
- DELETE Request: Used to remove existing information or records from a web server (e.g., deleting a photo or account).

Lab Questions Reference
-What method would be used to create a new user account?  
```POST```  
- What method would be used to update your email address?  
```PUT```  
- What method would be used to remove a picture you've uploaded to your account?  
```DELETE```  
- What method would be used to view a news article?  
```GET```

## Module 4: HTTP Status Codes
1. Status Code Ranges Overview
When a web server responds to a request, the first line always contains a 3-digit status code that informs the client of the outcome. These codes are divided into 5 numerical ranges:
- 100–199 (Informational Response): Informs the client that the initial part of their request was accepted and they should continue sending the rest. (Rarely used modernly)
- 200–299 (Success): Confirms that the client's request was processed successfully.
- 300–399 (Redirection): Redirects the browser to a different resource or webpage.
- 400–499 (Client Errors): Indicates that something went wrong on the client side (e.g., bad request, missing page, or lacking credentials).
- 500–599 (Server Errors): Indicates a failure on the server side while handling the request.

2. Common HTTP Status Codes Breakdown
- 200 - OK: Request completed successfully.
- 201 - Created: A new resource was successfully created (e.g., a new user or blog post).
- 301 - Moved Permanently: Permanently redirects the browser/search engine to a new web location.
- 302 - Found: Temporarily redirects the browser to another page.
- 400 - Bad Request: Request was malformed or missing required parameters expected by the server.
- 401 - Not Authorised: Requires user authentication (logging in) before access is granted.
- 403 - Forbidden: Permission denied regardless of whether the user is logged in or not.
- 404 - Page Not Found: The requested resource/page does not exist on the server.
- 405 - Method Not Allowed: The endpoint does not support the HTTP method used (e.g., sending a GET when POST was expected).
- 500 - Internal Server Error: An unexpected server-side error occurred (e.g., application code crash or database failure).
- 503 - Service Unavailable: Server is down due to maintenance or handling high traffic loads.

Lab Questions Reference
- What response code might you receive if you've created a new user or blog post article?
```201```
- What response code might you receive if you've tried to access a page that doesn't exist?
```404```
- What response code might you receive if the web server cannot access its database and the application crashes?
```500``` ```(Note: In the screenshot input box it shows 503, but standard crashes are 500)```
- What response code might you receive if you try to edit your profile without logging in first?
```401```

## Module 5: Headers
1. Overview
Headers are key-value pairs sent alongside HTTP requests and responses to pass metadata between the client and server. While no specific headers are strictly mandatory to construct a basic request, they are essential for rendering web applications correctly.
2. Core Headers Breakdown
Common Request Headers (Sent from client to server)
- Host: Tells the server which exact website/domain is being requested when a single IP hosts multiple sites.
- User-Agent: Identifies client browser and OS specifications so the server can return compatible markup.
- Content-Length: Informs the server of the exact size (in bytes) of incoming payload data (e.g., POST form submissions).
- Accept-Encoding: Specifies supported compression methods (e.g., gzip) to minimize bandwidth.
- Cookie: Passes stored session identifiers and state data back to the server.

Common Response Headers (Sent from server to client)
- Set-Cookie: Sends state or session identifiers to the browser to store for subsequent requests.
- Cache-Control: Instructs the browser on how long it may store cached copies of resources locally.
- Content-Type: Defines the media format of the payload (HTML, CSS, JavaScript, JSON, image, etc.) so the client renders it properly.
- Content-Encoding: Indicates the compression algorithm used on the response body.

Lab Questions Reference
- What header tells the web server what browser is being used?  
```User-Agent```  
- What header tells the browser what type of data is being returned?  
```Content-Type```  
- What header tells the web server which website is being requested?  
```Host```

## Module 6: Cookies
1. Overview
Cookies are small pieces of data stored on a client's computer by the web browser. Because the HTTP protocol is stateless (it does not track or remember previous requests), cookies are used to persist state—allowing web servers to remember user identity, site preferences, or session status across multiple requests.
2. How Cookies Work
  1. Initial Request: The client makes a GET request to a webpage.
  2. 2. Initial Response: The server responds with the webpage (e.g., a form asking for a username).
  3. 3. Data Submission: The client submits form data (e.g., POST request with name=adam).
  4. Setting the Cookie: The server responds with a Set-Cookie: name=adam response header, instructing the client browser to store this data locally.
  5. Subsequent Requests: On every following request, the client's browser automatically sends the stored data back to the server in a Cookie: name=adam request header.
  6. State Recognition: The server reads the incoming cookie and personalizes the response (e.g., displaying "Welcome back adam") instead of asking for credentials again.

3. Common Uses & Inspection
- Authentication & Tokens: Cookies are most commonly used for session authentication. Instead of storing passwords in plaintext, servers store a complex, unguessable session token.
- Viewing Cookies: You can view active cookies in your browser using Developer Tools (F12) under the Network tab by selecting a request and inspecting its Cookies breakdown.

Lab Questions Reference
- Which header is used to save cookies to your computer?
```Set-Cookie```
