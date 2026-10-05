# Room Name: DNS in Detail
Difficulty: Easy
Category: Networking / Web Fundamentals
Platform: TryHackMe

### 🎯 Room Overview
The DNS in Detail room provides a foundational breakdown of how the Domain Name System (DNS) functions across the internet. It covers domain naming hierarchies, different DNS record types, the step-by-step lifecycle of a DNS resolution request, and practical query execution using the nslookup command-line utility.

### 📚 Technical Concepts Explained
#### Task 1: What is DNS?
DNS (Domain Name System) acts as the phonebook of the Internet. Computers communicate across networks using IP addresses (such as 104.26.10.229), which are difficult for humans to memorize. DNS automatically translates human-readable domain names (e.g., tryhackme.com) into their respective numerical IP addresses.

#### Task 2: Domain Hierarchy
Domains follow an inverted tree-structure reading right to left:
```bash
               . (Root)
               |
        .com / .edu / .org (TLD)
               |
          tryhackme (SLD)
               |
            admin (Subdomain)
```
1. Root Domain (```.```): The top level represented by an implicit dot at the end of every domain path.
2. Top-Level Domain (TLD): The rightmost part of a domain name.
   - gTLD (Generic TLD): Denotes site purpose (e.g., ```.com, .org, .edu```).
   - ccTLD (Country Code TLD): Denotes geographic origin (e.g., ```.co.uk, .ca, .in```).
3. Second-Level Domain (SLD): The core brand/name registered directly under a TLD (e.g., ```tryhackme``` in ```tryhackme.com```).
4. Limits: Max 63 characters, ```a-z```, ```0-9```, and hyphens ```-``` (cannot start or end with a hyphen).
5. Subdomain: Sits to the left of the SLD to divide administrative areas or services (e.g., ```admin``` in ```admin.tryhackme.com```).
   - Limits: Total length of full domain name cannot exceed 253 characters.

#### Task 3: DNS Record Types
DNS information is categorized using standard record types:
- ```A``` Record: Maps a domain name directly to an IPv4 address (e.g., ```104.26.10.229```).
- ```AAAA``` Record: Maps a domain name directly to an IPv6 address (e.g., ```2606:4700:20::681a:be5```).
- ```CNAME``` Record (Canonical Name): Aliases one domain name to another canonical domain name (e.g., ```store.tryhackme.com $\rightarrow$ shops.shopify.com```).
- ```MX``` Record (Mail Exchange): Directs email queries to specified mail servers alongside a numeric priority value (lower numbers take higher priority).
- ```TXT``` Record: Stores arbitrary text strings. Used for domain verification, SSL validation, and email authentication standards (SPF, DKIM, DMARC).

#### Task 4: Making a DNS Request
When requesting a web address, the client goes through a multi-tier query process:
```bash
[Client Computer] ──(1)──> [Recursive DNS Server (ISP/Public)]
                                     │
                                   (2)──> [Root DNS Server]
                                     │
                                   (3)──> [TLD DNS Server]
                                     │
                                   (4)──> [Authoritative DNS Server]
```
1. Local & Recursive Lookup: The client queries local cache first. If absent, the request goes to the Recursive DNS Server (typically provided by the ISP or configured DNS resolvers like ```8.8.8.8```).
2. Root Server (```.```): Directs the recursive server to the appropriate TLD server (.com, .org, etc.).
3. TLD Server: Directs the query to the specific Authoritative DNS Server responsible for the target domain.
4. Authoritative Server: Stores actual zone records and returns the final destination IP.
5. Caching & TTL: Responses are stored locally for the duration specified by the TTL (Time To Live) value in seconds to minimize network redundancy.

#### 🧪 Task 5: Practical Exercise & Query Logs
Below are the commands executed using ```nslookup``` during the lab session to extract specific record values:
1. CNAME Query
Querying alias details for ```shop.website.thm```:
```bash
nslookup --type=CNAME shop.website.thm
```
- Output Answer: ```shops.myshopify.com```

2. TXT Record Query
Extracting verification text strings from ```website.thm```:
```bash
nslookup --type=TXT website.thm
```
- Output Answer: ```THM{7012BBA60997F35A9516C2E16D2944FF}```

3. MX Record Query
Checking priority value assigned to the mail exchanger:
```bash
nslookup --type=MX website.thm
```
- Output Answer: ```Priority value is 30 (Server target: alt4.aspmx.l.google.com)```

4. A Record Query
Retrieving IPv4 binding for ```www.website.thm```:
```bash
nslookup --type=A www.website.thm
```
- Output Answer: ```10.10.10.10```

### 💡 Key Takeaways
Reconnaissance Utility: Understanding DNS record structure is essential during active/passive reconnaissance to map target subdomains, mail infrastructure, and third-party SaaS integrations.
Enumeration Tooling: Command-line utilities like ```nslookup```, ```dig```, and ```host``` are core tools for querying nameservers directly during security assessments.
