---
title: BioHazard
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 25
---

# BioHazard

<img width="912" height="251" alt="image" src="https://github.com/user-attachments/assets/b129a7f8-802a-45e4-b410-d57f87090a4a" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> A CTF room based on the old-time survival horror game, Resident Evil. Can you survive until the end?

## Recon

As always, I started with an nmap scan which revealed services on ```ports 21, 22, and 80```.

Visiting the webserver, we are greeted with the start of a story.

<img width="1714" height="799" alt="image" src="https://github.com/user-attachments/assets/f2aba674-85b4-4824-b3e1-2443f0454cb8" />

Clicking the hyperlink, we are redirected to this page with the subdir to visit in the source code: ```/diningRoom```.

Visiting the ```/diningRoom``` subdir we see the continuation of the story as well as this b64 encoded string in the source code: ```SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=```. Decoded, it translates to: ```How about the /teaRoom/```. 

On the page we see an option to take an emblem, going there we get an emblem flag and are instructed to reload the diningRoom page.

<img width="1848" height="773" alt="image" src="https://github.com/user-attachments/assets/a15f1478-f730-4358-8f97-b455971c885b" />

Submitting the flag, 










### priv esc



---

## Solution Steps

1. Enumerate the web server
2. Enumerate the ```/joomla``` subdir
3. Find the ```sar2html``` tool in the ```/_test``` subdir
4. Discover the rce exploit for ```sar2html```
5. exploit it and get creds form log.txt
6. establish an ssh session with creds
7. escalate privileges using ```find``` binary by executing the command ```find . -exec /bin/sh -p \; -quit```



## Thoughts
I am really enjoying these medium rated boxes. For my skill level, tehy are not too hard, yet not too easy. As always, I love exploiting SUID binaries simply becauseof how satisfying ti is to see the root shell prompt appear. But overall an interesting box. Exposed me to the ```sar2html``` rce exploit which was super simple yet interesting to learn about.

---

Thank you for reading!

