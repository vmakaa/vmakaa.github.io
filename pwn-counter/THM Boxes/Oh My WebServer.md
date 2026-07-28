---
title: Oh My WebServer
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 33
has_children: true
---

# Oh My WebServer

<img width="946" height="274" alt="image" src="https://github.com/user-attachments/assets/88d1c484-8984-49fa-ad93-aa7588454ba9" />

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Can you root me?

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 22 and 80```.

Opening up the webpage being served on ```port 80```, we see a landing page with no working functionality. Great.

I tried gobuster but all I found was an exposed directory listing for the ```/assets``` subdirectory which gave me literally no useful information, and I wasnt getting any hits on vhosts enumeration.

As a hail mary, I tried using nuclei to check out the web server and nuceli came in clutch by finding that the server was vulnerable to CVE-2021-41773. 

I found a [POC](https://github.com/blackn0te/Apache-HTTP-Server-2.4.49-2.4.50-Path-Traversal-Remote-Code-Execution/blob/main/exploit.py) online and immediately got a shell. Easiest privilege escalation ever.

<img width="957" height="177" alt="image" src="https://github.com/user-attachments/assets/61ebffff-d2f1-4f0c-9ee2-8fade08b6e3a" />

### Shell Foothold

The shell that the script provided was pretty bad so I used it to get a more stable reverse shell using a python reverse shell. 

Enumerating around the shell I found a root owned ```.dockerenv``` file, but couldn't find the usual low hanging fruits, so I turned to linpeas.

After running linpeas, I found an interesting find: a python capability.

<img width="352" height="58" alt="image" src="https://github.com/user-attachments/assets/c6ae7f27-e6bb-4394-b5b5-b600e823b8c4" />

### Initial priv esc

Using this information, I searched online for [how to exploit this](https://www.hackingarticles.in/linux-privilege-escalation-using-capabilities/), and it was as simple as a command one liner since this capability allowed us to set the uid. 

I executed ```python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'``` and got root.

<img width="721" height="203" alt="image" src="https://github.com/user-attachments/assets/3ecb9526-30b2-4b52-aea6-90d7a47eff7e" />

Now I can check out that docker file.

### Docker Break Out

The ```.dockerenv``` file was empty unfortunately but it provided me with the crucial knowledge that I am in a container.

Enumerating around, I didnt find much. The docker command wasnt even on the box.

---

After some googling, I realized that I was on a 'docker network' with the host machine as a container, meaning that I could try and scan the host machine if they have any ports open we could possibly exploit.

I downloaded the static binary for nmap over to the box and scanned the host machine (172.17.0.1).

They had an interesting port open: ```port 5986```.

After some googling, I found that this port was used by Azure to manage linux servers as part of their commercial OMI. I also immediately found an exploit for it.

I moved the exploit to the box and tested the rce script with the parameter id, and it worked nice.

I then put a reverse shell in the command parameter, set up a listener, and got a reverse shell as the host machine. Amazing sauce.

<img width="1877" height="284" alt="image" src="https://github.com/user-attachments/assets/0f159f14-6493-4d3b-9a43-0f2380ce2b0b" />

---

## Solution Steps

1. Use nuceli to find the rce vulnerability on the web server
2. get a shell and escalate to root uid using python set uid capabilities
3. break out the docker container using OMI on port 5986 exploit to get reverse shell as machine host


## Thoughts

This was a really fun box. Even though the initial priv esc was really easy, I found breaking out the docker container to be a niche and cool path to learn about and add to my toolbelt. Definitely Recommend. 

---

Thank you for reading!

