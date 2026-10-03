# TryHackMe: What is Networking? — Complete Walkthrough & Notes
## Task 1: What is Networking?
### Overview
A network is a collection of connected devices or systems that share resources, data, or communication pathways. Networks exist in daily human infrastructure—such as electrical power grids, postal delivery routes, and public transport systems—as well as in computing environments.   

In computing, a network can consist of as few as 2 devices connected locally or billions of devices communicating globally across the Internet.   

#### Questions & Answers
```bash
Question	                                                            Answer
What is the key term for devices that are connected together?	        Network
```
## Task 2: What is the Internet?
### Overview
The Internet is a global "network of networks". It connects millions of private, public, academic, business, and government networks together to facilitate global communication and resource sharing.   

### Historical Milestones
- ARPANET: Developed in the late 1960s with funding from the U.S. Department of Defense, ARPANET served as the first operational packet-switching network and the predecessor to the modern Internet.
- World Wide Web (WWW): Invented in 1989 by Tim Berners-Lee, the WWW introduced an information system where documents and web resources are identified by URLs and accessible via the Internet.

### Types of Networks
- Private Network: A restricted network accessible only to authorized devices (e.g., home Wi-Fi networks or internal corporate LANs).
- Public Network: An open infrastructure that connects smaller networks together to allow global communication (e.g., the Internet).

#### Questions & Answers
```bash
Question	                           Answer
Who invented the World Wide Web?	   Tim Berners-Lee
```
## Task 3: Identifying Devices on a Network
To route traffic and establish communication, every device on a network must possess identifiable network labels: IP Addresses (logical identification) and MAC Addresses (physical hardware identification).   
1. IP Addresses (Internet Protocol)
An IP address is a logical numerical label assigned to each device connected to a network that uses the Internet Protocol for communication.
- IPv4 Structure: Consists of 4 octets separated by dots (e.g., 192.168.1.1). Each octet ranges from 0 to 255. IPv4 uses a 32-bit addressing scheme, yielding approximately $4.29 \text{ billion}$ ($2^{32}$) total unique addresses.
- IPv6 Structure: Created to address the global exhaustion of IPv4 addresses. IPv6 uses a 128-bit hexadecimal addressing format ($2^{128}$ total addresses).
- Public vs. Private IP Addresses:
  - Private IP: Assigned to devices within a local network by a router via DHCP.
  - Public IP: Assigned to the gateway/router by the Internet Service Provider (ISP) to identify the network on the public Internet.

2. MAC Addresses (Media Access Control)
A MAC address is a unique physical address permanently assigned to a device's Network Interface Card (NIC) at the factory.
- Format: Consists of 12 hexadecimal characters split into 6 pairs separated by colons or hyphens (e.g., a4:c3:f0:85:ac:2d).
- Structure:
  - First 6 characters (OUI): Identify the manufacturer/vendor (e.g., Intel, Cisco).
  - Last 6 characters: Unique identifier for the specific NIC.
- MAC Spoofing: The practice of altering the source MAC address sent in network requests to impersonate another device. Attackers use spoofing to bypass access controls, network filters, or captive portal paywalls.

 #### Questions & Answers
 ```bash
Question                                                                                                                    Answer
What does the term "IP" stand for?                                                                                          Internet Protocol
What is each section of an IP address called?                                                                               Octet
How many sections (in digits) does an IPv4 address have?                                                                    4
What does the term "MAC" stand for?                                                                                         Media Access Control
Deploy the interactive lab using the "View Site" button and spoof your MAC address to access the site. What is the flag?    THM{YOU_GOT_ON_TRYHACKME}
```
## Task 4: Ping (ICMP)
### Overview
```ping``` is a foundational network diagnostic utility used to test the reachability of a host on an IP network and measure the round-trip time (RTT) for messages sent from the originating host to a destination computer.   
- Protocol: Uses ICMP (Internet Control Message Protocol).
- Mechanism: The source host sends an ICMP Echo Request packet to the destination host, which responds with an ICMP Echo Reply.
- Basic Syntax:
```bash
ping <IP_address_or_Domain>
```
- Example Commands:
  - Ping an IPv4 address: ```ping 192.168.1.254```
  - Ping with packet count limit (Linux): ```ping -c 4 8.8.8.8```

#### Questions & Answers
```bash
Question                                              AnswerWhat
protocol does ping use?                               ICMP
What is the syntax to ping 10.10.10.10?               ping 10.10.10.10
What flag do you get when you ping 8.8.8.8?           THM{I_PINGED_THE_SERVER}
```
## Task 5: Continue Your Learning: Intro to LAN
### Overview
This task concludes the foundational What is Networking? room and directs learners to the next module in the Pre-Security path: Intro to LAN (Local Area Networks), where topics such as subnetting, switches, routers, and ARP are explored further.
#### Questions & Answers
```bash
Question                                                Answer
Read the above and continue your networking journey!    No response required
```
