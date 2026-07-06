---
title: Bounty Hacker
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 18
---

# Bountry Hacker

<img width="986" height="345" alt="THM-BH_badge" src="https://github.com/user-attachments/assets/00bf0c4b-fbf6-4c10-b495-76ec34dc22fc" />

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> A Cowboy Bebop themed box where you must prove you are the greatest hacker in the galaxy.

## Recon
As always I start off with ```nmap -sC -sV -vv ip``` on the target box. This scan revealed that the target box has a webserver running on ```port 80```, ssh running on ```port 22```, and ftp running on ```port 21``` with anonymous login available. 

Visiting the webpage via Firefox, we are greeted with a request from the crew to gain root access to a system. 

<img width="1909" height="871" alt="image" src="https://github.com/user-attachments/assets/e26c49f2-6ba7-472c-9983-f81a67d67cef" />

Given that this is a web server, I run a gobuster scan in the background while enumerating to hopefully gain some usefuil info using ```gobuster dir http://ip -w /usr/share/worlists/dirbuster/small.txt```.

I then logged into the frp service and found two interesting files: ```locks.txt``` & ```task.txt```.

<img width="363" height="525" alt="image" src="https://github.com/user-attachments/assets/8586995c-05b8-471a-8745-ce8e2b1798e5" />

The output of ```locks.txt``` appears to be a password list and there is a potential username at the end of ```task.txt```.

Gobuster did not come back fruitful and the only service with a login on teh box was ssh, so I decided to pull on this thread to see where it led.

I set up hydra to try and brute force the ssh login with the username ```lin``` and the worlist of ```locks.txt``` using ```hydra -l lin -P locks.txt ssh://ip -t 4 -v``` and I got a valid pair of credentials! ヾ(≧▽≦*)o

<img width="1656" height="244" alt="Screenshot 2026-06-30 164523" src="https://github.com/user-attachments/assets/fa73f4d2-762f-4b3a-898a-814f7151c46a" />

## Priv Esc
Using these credentials, I logged into ssh and looked for any suid binaries with ```find / /perm -u=s 2>/dev/null | grep /usr/```. After cross referencing a couple of the binaries with GTFObins, I remembered that I still hadn't performed ```sudo -l``` so I did that and found that I could run ```/bin/tar``` as ```root``` on this machine.

I searched GTFObins for this binary and found an entry to reference.

<img width="979" height="481" alt="image" src="https://github.com/user-attachments/assets/7b6e0196-649c-40f8-ab10-24f48f08917a" />

Using this, I did ```sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh```, got a root shell, and pwned the box.

## Solution Steps

1. Brute force ssh login with info from ftp .txt files.
2. ```sudo -l``` to find /bin/tar can be ran as root
3. privesc with ```sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh```

## Thoughts
I thought this was a pretty fun box to pwn, I always find it satisfying to suddenyl see the root shell prompt. Even though this was an easy box, I am still proud of myself because this is the first box I completed without having to refernce a writeup!

--

Thank you for reading!

