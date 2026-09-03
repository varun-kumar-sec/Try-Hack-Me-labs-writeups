# TryHackMe: Vulnerabilities 101 Write-up

**Room Link:** [TryHackMe - Vulnerabilities 101](https://tryhackme.com/room/vulnerabilities101)  
**Difficulty:** Easy  
**Category:** Fundamentals / Vulnerability Management  

---

## Room Overview
This room introduces the fundamentals of security vulnerabilities, how they are categorized, scoring systems used to measure risk, public vulnerability databases, and practical methods for discovering exploits using version disclosure.

---

## Task 2: Introduction to Vulnerabilities

A **vulnerability** is a weakness or flaw in a system's design, implementation, or behavior that an attacker can exploit to perform unauthorized actions or gain unauthorized access.

### 5 Main Vulnerability Categories
* **Operating System:** Flaws in OS code (e.g., Windows/Linux bugs) that often lead to privilege escalation.
* **(Mis)Configuration-based:** Security issues stemming from improperly configured applications or services.
* **Weak or Default Credentials:** Applications using factory-default or weak logins (e.g., `admin:admin`).
* **Application Logic:** Flaws in application design or authentication mechanisms (e.g., cookie manipulation).
* **Human-Factor:** Exploiting human trust or behavior (e.g., phishing).

### Questions & Answers
* **An attacker has been able to upgrade the permissions of their system account from "user" to "administrator". What type of vulnerability is this?**
  * `Operating System`
* **You manage to bypass a login panel using cookies to authenticate. What type of vulnerability is this?**
  * `Application Logic`

---

## Task 3: Scoring Vulnerabilities (CVSS & VPR)

Vulnerability scoring assigns quantitative values to flaws to help organizations prioritize patching since only ~2% of vulnerabilities are ever exploited in the wild.

### Framework Comparison

| Feature | CVSS (Common Vulnerability Scoring System) | VPR (Vulnerability Priority Rating) |
| :--- | :--- | :--- |
| **Type** | Open-source / Free standard | Commercial / Proprietary (Tenable) |
| **Approach** | Static score based on impact & severity | Dynamic score based on real-time threat risk |
| **Scale** | None (0), Low (0.1-3.9), Med (4.0-6.9), High (7.0-8.9), Critical (9.0-10.0) | Low (0.0-3.9), Med (4.0-6.9), High (7.0-8.9), Critical (9.0-10.0) |

### Questions & Answers
* **What year was the first iteration of CVSS published?**
  * `2005`
* **If you wanted to assess vulnerability based on the risk it poses to an organisation, what framework would you use?**
  * `VPR`
* **If you wanted to use a framework that was free and open-source, what framework would that be?**
  * `CVSS`

---

## Task 4: Vulnerability Databases

Vulnerability databases keep track of security flaws, CVE identifiers, and practical exploit scripts across all software ecosystems.

### Core Terminology
* **Vulnerability:** Flaw or weakness in application design or system logic.
* **Exploit:** An action or tool that takes advantage of a vulnerability.
* **Proof of Concept (PoC):** Code or demonstration proving a vulnerability can be exploited.

### Key Resources
* **NVD (National Vulnerability Database):** Public directory listing all categorized CVEs (`CVE-YEAR-IDNUMBER`).
* **Exploit-DB:** Managed by OffSec, containing public exploit scripts and PoCs organized by software version.

### Questions & Answers
* **Using NVD, how many CVEs were published in July 2021?**
  * `1554`
* **Who is the author of Exploit-DB?**
  * `OffSec`

---

## Task 5: An Example of Finding a Vulnerability

Demonstrates how attackers pair minor information disclosure flaws with exploit databases to find high-impact attacks.

### Workflow
1. **Version Disclosure:** Identifying the application version (e.g., `Apache Tomcat 9.0.17`).
2. **Database Lookup:** Searching Exploit-DB for known vulnerabilities linked to that version.
3. **Exploit Selection:** Choosing an exploit (e.g., Remote Code Execution) targeting the specific release.

### Questions & Answers
* **What type of vulnerability did we use to find the name and version of the application in this example?**
  * `Version Disclosure`
