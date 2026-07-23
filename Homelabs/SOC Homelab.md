---
title: SOC Homelab
parent: Homelabs
nav_order: 1
---

# Homelab Description
> This is all the information for the homelab I made and presented on to the LSU security society. This homelab takes a cloud-deployed honeypot and uses Suricata, TheHive (Incident Management), EveBox (Quick alert viewing), and tailscale to connect all Virtual Machines together.

# Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Tech Stack
I decided on a fully cloud-based approach using [DigitalOcean](https://www.digitalocean.com/).

I had one virtual machine running the [SANS ISC D-Shield Honeypot](https://isc.sans.edu/honeypot.html), which included honeypot services for ssh, telnet, and also had a web instance running to capture web attacks.

In a separate VM, I installed Suricata with all the latest rules and also installed a lightweight eve.json visualization program called Evebox. On the same machine, I also deployed a docker container of [TheHive](https://docs.strangebee.com/thehive/installation/docker/), an open source Incident Management System.

My vision was to have Suricata monitor inbound and outbound traffic, and when Suricata would alert on malicious traffic the data of thta alert could be viewed in the nice GUI provided by Evebox. For Incident Management, I had Claude write me a python script that would periodically fetch eve.json and import them to TheHive as alerts so that an alert may be escalated to a case.

## Technical Details
In order for Suricata to have visbility on ingress and egress traffic coming from the honeypot, I created a two gre tunnels, one from the suricata machine to the honeypot (Tunnel A) and vice versa (Tunnel B). I then used iptables to send a copy of all inbound and outbound traffic thrught the IP address of tunnel B.

### Routing honeypot traffic to Suricata

The following two commands are needed on the honeypot to forward all of its inbound and outbound traffic:

iptables -t mangle -A PREROUTING -j TEE --gateway IP of Tunnel B

iptables -t mangle -A POSTROUTING -j TEE --gateway IP of tunnel B

---

The following commands are how to set up a GRE link between two machines, this is essential so that the honeypot has an interface to send its copied traffic to:

Machine1 (M1):

ip tunnel add gre1 mode gre remote <M2 IP> local <M1 IP> ttl 255
ip link set gre1 up
ip addr add 172.16.0.1/30 dev gre1

Machine2 (M2):

ip tunnel add gre1 mode gre remote <M1 IP> local <M2 IP> ttl 255
ip link set gre1 up
ip addr add 172.16.0.2/30 dev gre1

# Alert Analysis
Before analyzing the payload of the alert, I first check the IP against OSINT tools like [AbuseIPDB](https://www.abuseipdb.com/), [Spur](https://spur.us/), and [Shodan](https://www.shodan.io/) to collect information like if the IP has been reported for malicious activity prior, is this IP using a VPN or a proxy, and what type of machine is this IP originating from and what ports are open.

Then, I take a look at the Suricata rule that alerted on the traffic and make sure I understand why it had alerted to help me determine the liklihood of the alert being a false positive depending on how specifc the rule is.

I also check the amount of bytes sent to the client (usually the attacker) and to the server (usually my honeypot). Although most of the traffic hitting my honeypot was most likely facilitated using an automated internet ipv4 scanner, seeing that the client recieved nothing while your server recieved a large amount of bytes adds onto the reasoning of scanning behavior.

Finally, I look at the payload itself. For example in a RCE attempt, I usually see attackers attempting to make the honeypot download a script from a server, likely using simple http server, and depending on the language of the script, will then compile and try ot execute the script. In one instance, the attacker did not even bother renaming the script and OSINT found that the attacker was trying to run [Fish Shell](https://fishshell.com/), a shell which has a  selling point of nice color highlighting, which was pretty funny.

---

Over the honeypot's uptime, I have seen some interesting alerts ranging from fairly new like React2Shell all the way to fairly old like Realtek SDK RCE.

Getting my hands dirty and setting up a honeypot was good learning experience and provided me insight that I was excited I would be able use at my SOC analyst internship at C3 to quickly detect tattacks that are targeted vs automated, which made my case analysis time twice as fast.

## Future
I doubt this will be my last experience with honeypots, as I really want to try two new honeypots that I had found. One of them is called [Gaspot](https://github.com/sjhilt/GasPot), which is an ICS honeypot that "emulates Veeder-Root TLS-350 / TLS-450 Automatic Tank Gauge (ATG) controller commonly found at gas stations worldwide." Another one that I found is actually a list of honeypots all running together on a single amchine developed by T-Mobile's Telekom Security. This honeypot is called [T-Pot](https://github.com/telekom-security/tpotce) and includes honeypots that mimic LLMs, ICSs, and classic ssh and web honeypots just to name a few.

## Screenshots

### 66 React2Shell attempts

Here are the total number of critical alerts I have amassed throughout the uptime of my honeypot, as you can see one scanner hit my honeypot trying to use the React2Shell exploit 66 times!

<img width="1919" height="991" alt="Screenshot 2026-05-04 114842" src="https://github.com/user-attachments/assets/e2615c3c-d5f2-4e89-afa4-795663a2fabb" />

### Realtek SDE RCE Attempt

Here is a close up of the network data from a Realtek SDE RCE attempt:

<img width="1885" height="268" alt="Screenshot 2026-05-04 114948" src="https://github.com/user-attachments/assets/04310285-46b7-4788-9fe3-49f1c6ca457f" />

#### Realtek SDE RCE Payload

and here is the payload for that attempt:

<img width="925" height="110" alt="Screenshot 2026-05-04 115012" src="https://github.com/user-attachments/assets/7602df02-00db-4767-b772-8a8a315910bd" />

### SANS DShield Honeypot

Deploying the SANS ISC D-Shield Honeypot directly supports the SANS threat intel research and as a benefit you get access to their threat intel. It also has a web GUI for data that it periodically fetches from your honeypot to statistically display attacks and reports. The following are screenshots of the data and report charts:

#### Total Logs Recorded during Honeypot Uptime

<img width="1123" height="302" alt="Screenshot 2026-05-04 115218" src="https://github.com/user-attachments/assets/11522887-9845-4009-8e68-e7c8017d6e5b" />

#### Top IP Addresses Sending Traffic to Honeypot

<img width="619" height="351" alt="image" src="https://github.com/user-attachments/assets/ec6bf2dc-381c-45df-a395-a724d42dfaa3" />

#### Top Usernames used in Brute Force Attempts

<img width="541" height="274" alt="image" src="https://github.com/user-attachments/assets/a5e66d6e-5c3b-4abe-9927-ac6a94f4775a" />

#### Top Passwords Used in Brute Force Attempts

<img width="502" height="286" alt="image" src="https://github.com/user-attachments/assets/d09d51f7-67df-437d-b7c4-8bd1e76df8e5" />

### Firewall logs

The following two screenshots are from the firewall logs

#### Highest number of logs during Honeypot Uptime

<img width="1057" height="187" alt="Screenshot 2026-05-04 115457" src="https://github.com/user-attachments/assets/aad6029e-f371-4918-88e9-85195e705376" />

#### Firewall dashboard

<img width="1029" height="870" alt="Screenshot 2026-05-04 115525" src="https://github.com/user-attachments/assets/62ad020f-d0cd-4398-8b35-a039e30a4935" />

_________________________________________

















