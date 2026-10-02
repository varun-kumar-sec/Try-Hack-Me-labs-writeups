# Introductory Networking: Deep Dive & Technical Walkthrough1.

1. The OSI Model (Open Systems Interconnection)
The OSI model is a 7-layer conceptual framework used to standardize how applications communicate over a network. Each layer handles a specific task and passes data to the layer above or below it.

```bash
+-------------------------------------------------------------+
| Layer 7: Application  (User-facing protocols: HTTP, SSH)    |
| Layer 6: Presentation (Data formatting, encryption)         |
| Layer 5: Session      (Connection tracking & control)       |
| Layer 4: Transport    (End-to-end delivery: TCP/UDP)        |
| Layer 3: Network      (Logical routing & IP addressing)     |
| Layer 2: Data Link    (Physical addressing & MAC frames)    |
| Layer 1: Physical     (Raw bitstream transmission)          |
+-------------------------------------------------------------+
```
### Layer-by-Layer Breakdown
- Layer 7 — Application: The point where end-user software interacts with network capabilities. Web browsers, SSH clients, and email protocols live here.
- Layer 6 — Presentation: Acts as a data translator. It ensures data from the application layer is properly formatted, serialized, compressed, or encrypted (e.g., converting text to UTF-8 or parsing JSON/XML).
- Layer 5 — Session: Manages, maintains, and terminates communication channels between applications. It tracks active connections so data streams do not mix.
- Layer 4 — Transport: Manages data flow and end-to-end delivery. It splits long streams into smaller chunks and selects the protocol:
  - TCP (Transmission Control Protocol): Connection-oriented, reliable, orders packets, and retransmits lost data via ACKs.
  - UDP (User Datagram Protocol): Connectionless, fast, and does not check for lost packets (ideal for streaming or DNS).
- Layer 3 — Network: Determines how data is routed across different networks. It uses logical IP addressing (IPv4/IPv6) and ICMP. Routers operate at this layer.
- Layer 2 — Data Link: Transfers data between devices on the same physical network using MAC (Media Access Control) addresses. Switches operate at this layer.
- Layer 1 — Physical: Converts binary data into physical pulses—signals sent via Ethernet copper cables, fiber optics, or radio waves.

### Memory Aid (Mnemonic)
```An Excellent Swimmer Goes To Save People or Please Do Not Throw Sausage Pizza Away```.

### Task 2 Questions & Answers
```bash
Question                                                                           Answer
Which layer would choose to use TCP or UDP?                                        Transport
Which layer checks received packets to make sure that they haven't been damaged?   Transport
Which layer would handle formatting data (e.g., to JSON or XML)?                   Presentation
Which layer is responsible for routing data across different networks?             Network
Which layer is the lowest layer to interact directly with software applications?   Application
What is the primary protocol used at the Network layer for addressing?             IP
Which layer uses MAC addresses to identify devices on a local network?             Data Link
Name one of the physical mediums used to transmit data at Layer 1.                 Ethernet
```
2. Encapsulation & De-encapsulation
As data moves through the OSI model, each layer adds its own metadata (header) to the data unit. This wrapping process is Encapsulation. When the destination device receives the raw bits, it strips the headers layer-by-layer—a process known as De-encapsulation.
```bash
[Data]                                    -> Application / Presentation / Session
[Transport Header | Data]                 -> Transport Layer (Segment / Datagram)
[IP Header | Transport Header | Data]     -> Network Layer (Packet)
[MAC Header | IP | Transport | Data | Trailer] -> Data Link Layer (Frame)
101010100110101...                        -> Physical Layer (Bits)
```
### Protocol Data Units (PDUs)
Each layer has a specific name for its wrapped data chunk:   
```bash
Layer Range                    Protocol Data Unit (PDU)
Layers 7–5                     Data
Layer 4                        Segment (TCP) / Datagram (UDP)
Layer 3                        Packet
Layer 2                        Frame (includes header + trailer for error checks)
Layer 1                        Bits (raw binary stream)
```
### Task 3 Questions & Answers
```bash
Question                                                                                                   Answer
What is the name of the Data Link layer PDU?                                                               Frame
What is the name of the Network layer PDU?                                                                 Packet
What is the name of the Transport layer PDU when using TCP?                                                Segment
What is the name of the Transport layer PDU when using UDP?                                                Datagram
What is the process called where headers and trailers are added to data as it moves down the OSI layers?   Encapsulation
```
3. The TCP/IP Model & The TCP 3-Way Handshake
While the OSI model is a 7-layer theoretical framework, the internet relies on the 4-layer TCP/IP Model.

### OSI vs. TCP/IP Layer Mapping 
```bash
TCP/IP                 LayerCorresponding OSI Layers              Protocols / Concepts
Application            Application, Presentation, Session         HTTP, SSH, DNS, FTP
Transport              Transport                                  TCP, UDP
Internet               Network                                    IP, ICMP, ARP
Network Interface      Data Link, Physical                        Ethernet, Wi-Fi, MAC addresses
```
### The TCP 3-Way Handshake
Before TCP transmits application data, it establishes a reliable connection using three specific flags:   
```bash
Client                                      Server
  |                                           |
  | ------- 1. SYN (Synchronise) ---------->  |  (Client requests connection)
  |                                           |
  | <------ 2. SYN-ACK (Sync-Ack) ----------  |  (Server acknowledges & requests back)
  |                                           |
  | ------- 3. ACK (Acknowledge) ---------->  |  (Client confirms, connection OPEN)
  v                                           v
```
1. SYN (Synchronise): The client sends a packet with the SYN flag set to request a connection and negotiate sequence numbers.
2. SYN/ACK: The server responds with SYN/ACK to acknowledge the client's request and send its own initial sequence number.
3. ACK (Acknowledge): The client responds with ACK to confirm receipt. The connection is now established.

### Task 4 Questions & Answers   
```bash
Question                                                                                                                                                  Answer
Which model was introduced first, OSI or TCP/IP?                                                                                                          TCP/IP
Which layer of the TCP/IP model covers the functionality of the Transport layer of the OSI model? (Full Name)                                             Transport
Which layer of the TCP/IP model covers the functionality of the Session layer of the OSI model? (Full Name)                                               Application
The Network Interface layer of the TCP/IP model covers the functionality of two layers in the OSI model. These layers are Data Link, and?.. (Full Name)   Physical
Which layer of the TCP/IP model handles the functionality of the OSI network layer?                                                                       Internet
What kind of protocol is TCP?                                                                                                                             Connection-based
What is SYN short for?                                                                                                                                    Synchronise
What is the second step of the three way handshake?                                                                                                       SYN/ACK
What is the short name for the "Acknowledgement" segment in the three-way handshake?                                                                      ACK
```
4. Networking Tools Walkthrough
### A. Ping (ICMP Echo)   
```ping``` sends ICMP Echo Request packets to test if a remote host is reachable and measures latency.      
```bash
# Basic syntax
ping <target>

# Common switches
ping -4 bbc.co.uk    # Force IPv4 addressing
ping -i 2 google.com # Change request interval to 2 seconds
ping -v google.com   # Verbose output
```
- OSI Layer: Operates at Layer 3 (Network) via ICMP.
- Core Advantage: Provides a quick way to discover the underlying IP address behind a domain name.

### Task 5 Questions & Answers
```bash
Question                                                          Answer
What command would you use to ping the bbc.co.uk website?         ping bbc.co.uk
Ping muirlandoracle.co.uk. What is the IPv4 address?              217.160.0.152
What switch lets you change the interval of sent ping requests?   -i
What switch would allow you to restrict requests to IPv4?         -4
What switch would give you a more verbose output?                 -v
```
### B. Traceroute (Path Discovery)
```traceroute``` maps the hop-by-hop path packets take across intermediate routers to reach a destination.   
```bash
# Linux (uses UDP by default)
traceroute tryhackme.com

# Key flags
traceroute -i eth0 target.com  # Specify interface
traceroute -T target.com       # Use TCP SYN requests instead of UDP
```
- How it Works: It sends packets with incrementing TTL (Time to Live) values starting at 1. Each router that drops a packet sends back an ICMP Time Exceeded message, revealing its IP address.
- OS Implementation Differences:Windows (tracert):
  - Uses ICMP by default.
  - Linux (traceroute): Uses UDP by default.
### Task 6 Questions & Answers
```bash
Question                                                                                            Answer
Use traceroute on tryhackme.com. Can you see the path your request has taken?                       No answer needed
What switch would you use to specify an interface when using Traceroute?                            -i
What switch would you use if you wanted to use TCP SYN requests when tracing the route?             -T
[Lateral Thinking] Which layer of the TCP/IP model will traceroute run on by default (Windows)?     Internet
```
### C. WHOIS (Domain Registration OSINT)
```whois``` queries Domain Registrar databases to reveal administrative and technical details about a domain registration.     
```bash
whois facebook.com
```
- Extracted Information: Registrant organization, contact emails, registration dates, expiry dates, and authoritative nameservers.
- Privacy Note: European domains often redact personal details due to GDPR restrictions.

### Task 7 Questions & Answers
```bash
Question                                                                                                Answer
Perform a whois search on facebook.com                                                                  No answer needed
What is the registrant postal code for facebook.com?                                                    94025
When was the facebook.com domain first registered? (Format: DD/MM/YYYY)                                 29/03/1997
Perform a whois search on microsoft.comNo answer needed   Which city is the registrant based in?        Redmond
[OSINT] What is the name of the golf course that is near the registrant address for microsoft.com?      Bellevue Golf Course
What is the registered Tech Email for microsoft.com?                                                    msnhst@microsoft.com
```
### D. Dig (Domain Name System Queries)
```dig``` (Domain Information Groper) performs manual DNS queries against recursive or authoritative DNS servers.   
```bash
# Query a specific DNS server (e.g., Cloudflare 1.1.1.1)
dig google.com @1.1.1.1
```
### How DNS Resolution Works (Order of Precedence)
1. Local Hosts File: Checked first (```/etc/hosts``` on Linux or ```C:\Windows\System32\drivers\etc\hosts on Windows)```.
2. Local DNS Cache: OS memory stores recent query results.
3. Recursive DNS Server: Typically provided by an ISP or public provider (e.g., Google ```8.8.8.8``` / ```8.8.4.4``` or Cloudflare ```1.1.1.1```).
4. Root Name Servers (```.```): Directs the query to the correct Top-Level Domain (TLD) server.
5. TLD Servers (```.com, .org, .co.uk```): Directs the query to the domain's authoritative server.
6. Authoritative Name Server: Holds the definitive DNS records for the domain and returns the IP.

### Understanding DNS TTL (Time To Live)
In a ```dig``` output, the TTL (second column in the ANSWER section) specifies how long (in seconds) a DNS record can be cached locally before requesting a fresh update.   
- Example: A TTL of ```86400``` means the record is valid in the cache for 24 hours ($86,400\text{ seconds}$).
### Task 8 Questions & Answers
```bash
Question                                                                                                                                   Answer
What is DNS short for?                                                                                                                     Domain Name System
What is the first type of DNS server your computer would query when you search for a domain?                                               Recursive
What type of DNS server contains records specific to domain extensions (i.e. .com, .co.uk)? Use the long version of the name.              Top-Level Domain
Where is the very first place your computer would look to find the IP address of a domain?                                                 Hosts File
[Research] Google runs two public DNS servers. One of them can be queried with the IP 8.8.8.8, what is the IP address of the other one?    8.8.4.4
If a DNS query has a TTL of 24 hours, what number would the dig query show?                                                                86400
```
