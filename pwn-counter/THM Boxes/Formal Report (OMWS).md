---
title: Formal Report
parent: Oh My WebServer
grand_parent: THM Boxes
nav_order: 1
---

# Findings Report

**Target:** Oh My WebServer (TryHackMe Boot-to-Root Assessment)

<table>
  <tr>
    <td><strong>Target Host</strong></td>
    <td>10.x.x.x (TryHackMe assigned IP)</td>
  </tr>
  <tr>
    <td><strong>Assessment Type</strong></td>
    <td>Black-box, unauthenticated web & container escape assessment</td>
  </tr>
  <tr>
    <td><strong>Assessor</strong></td>
    <td>Vanik</td>
  </tr>
  <tr>
    <td><strong>Report Date</strong></td>
    <td>28 July 2026</td>
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

This report documents a black-box penetration test performed against the "Oh My WebServer" host, a TryHackMe boot-to-root exercise. The objective of the assessment was to identify and exploit vulnerabilities that would allow an unauthenticated attacker to compromise the confidentiality, integrity, and availability of the target environment.

The assessment identified a critical chain of vulnerabilities beginning with an unauthenticated remote code execution flaw in an outdated Apache HTTP Server installation, escalating to root within the compromised container via a misconfigured Linux capability, and ultimately resulting in full compromise of the underlying Docker host through an exposed and vulnerable management service reachable only from within the container's network.

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
    <td>Unauthenticated RCE via Apache HTTP Server Path Traversal (CVE-2021-41773 / CVE-2021-42013)</td>
    <td>🔴 Critical</td>
    <td>9.8</td>
  </tr>
  <tr>
    <td>2</td>
    <td>Local Privilege Escalation via Misconfigured Python Binary Capabilities</td>
    <td>🟠 High</td>
    <td>7.8</td>
  </tr>
  <tr>
    <td>3</td>
    <td>Insufficient Network Segmentation Between Container and Docker Host</td>
    <td>🟡 Medium</td>
    <td>5.3</td>
  </tr>
  <tr>
    <td>4</td>
    <td>Unauthenticated RCE via Exposed OMI Service on Docker Host (CVE-2021-38647 "OMIGOD")</td>
    <td>🔴 Critical</td>
    <td>9.8</td>
  </tr>
</table>

Chained together, these findings resulted in full compromise of both the web-facing container and its underlying Docker host, from unauthenticated network access to root-level code execution on the host machine, with no valid credentials required at any stage.

---

## 2. Scope

<table>
  <tr>
    <td><strong>In-Scope Host</strong></td>
    <td>Web container: TryHackMe assigned IP &nbsp;|&nbsp; Docker host: 172.17.0.1 (discovered during assessment)</td>
  </tr>
  <tr>
    <td><strong>In-Scope Ports</strong></td>
    <td>22 (SSH), 80 (HTTP) on the container; 5986 (WinRM/OMI) discovered on the Docker host</td>
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

- Reconnaissance & Service Enumeration (Nmap, Gobuster, virtual host enumeration)
- Vulnerability Scanning (Nuclei)
- Exploitation of Web-Facing Vulnerabilities (public CVE proof-of-concept)
- Post-Exploitation & Local Privilege Escalation (LinPEAS, Linux capabilities abuse)
- Container Escape & Internal Network Pivoting (host-network reconnaissance, secondary CVE exploitation)

---

## 4. Detailed Findings

### 4.1 Unauthenticated RCE via Apache HTTP Server Path Traversal (CVE-2021-41773 / CVE-2021-42013)

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
    <td>Apache HTTP Server (versions 2.4.49–2.4.50)</td>
  </tr>
</table>

**Description**

Initial content discovery via Gobuster and virtual host enumeration returned no meaningful attack surface beyond a static landing page and an exposed `/assets` directory listing. A subsequent Nuclei scan against the web server identified that the target was running a version of Apache HTTP Server vulnerable to a known path traversal flaw (CVE-2021-41773), which, when combined with `mod_cgi` being enabled, allows arbitrary remote code execution (CVE-2021-42013).

**Proof of Concept**

A publicly available exploit script was used to confirm and weaponize the vulnerability:

```
python3 exploit.py -t <target_ip> -p <port> -c <cmd>
```

The exploit returned the output of the specified command and an interactive shell on first execution, confirming unauthenticated remote code execution against the web server.

**Impact**

This finding allows a fully unauthenticated remote attacker to execute arbitrary commands on the host serving the web application, providing full initial access with no prerequisites.

**Remediation**

- Upgrade Apache HTTP Server to a patched version (2.4.51 or later).
- Disable `mod_cgi` unless explicitly required by the application.
- Implement a Web Application Firewall (WAF) rule set capable of detecting path traversal patterns as defense-in-depth.

---

### 4.2 Local Privilege Escalation via Misconfigured Python Binary Capabilities

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>High</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H</code> (7.8)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td><code>/usr/bin/python3.x</code> (Linux capability: <code>cap_setuid</code>)</td>
  </tr>
</table>

**Description**

Following initial access, enumeration with LinPEAS identified that the `python3` binary had been assigned the `cap_setuid` Linux capability. This capability allows a process to change its effective and real user IDs, which when granted to a general-purpose interpreter such as Python enables any user able to execute the binary to trivially escalate to root.

**Proof of Concept**

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Execution of this one-line command returned a root shell within the container.

**Impact**

Any user or process capable of invoking the `python3` binary can escalate directly to root, fully undermining any privilege separation within the container.

**Remediation**

- Remove unnecessary Linux capabilities from general-purpose interpreters and binaries; `cap_setuid` should never be assigned to `python3` or similar interpreters.
- Regularly audit binaries for assigned capabilities using `getcap -r /` as part of hardening and configuration review.
- Apply the principle of least privilege and assumed breach to all container images used in production.

---

### 4.3 Insufficient Network Segmentation Between Container and Docker Host

<table>
  <tr>
    <td><strong>Severity</strong></td>
    <td>Medium</td>
  </tr>
  <tr>
    <td><strong>CVSS 3.1 Vector</strong></td>
    <td><code>AV:A/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N</code> (5.3)</td>
  </tr>
  <tr>
    <td><strong>Affected Component</strong></td>
    <td>Docker bridge network (host reachable at <code>172.17.0.1</code>)</td>
  </tr>
</table>

**Description**

After escalating to root within the container, the presence of a root-owned `.dockerenv` file confirmed the foothold was inside a Docker container rather than a standalone host. Further enumeration revealed that the container resided on a Docker bridge network configuration in which the underlying Docker host itself was reachable and scannable as a network peer (`172.17.0.1`), despite no legitimate application requirement for the container to reach the host's management interfaces.

**Proof of Concept**

A static Nmap binary was transferred into the container (no `docker` CLI or standard Nmap install was present) and used to scan the host:

```bash
./nmap -p- 172.17.0.1
```

The scan identified port 5986 as open on the Docker host itself, which was a management interface with no legitimate reason to be reachable from inside an application container.

**Impact**

This misconfiguration allowed an attacker who had already compromised the container to pivot directly to the Docker host's internal management surface, which was ultimately exploited for full host compromise (Finding 4.4). Proper network segmentation would have prevented this pivot entirely, independent of any vulnerability present in the exposed service itself.

**Remediation**

- Apply strict Docker network policies (e.g. `--internal` networks, custom bridge ACLs) so that application containers cannot reach host-level management interfaces.
- Bind host management services (such as OMI/WinRM) to specific host-only interfaces rather than all interfaces reachable from container networks.
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
