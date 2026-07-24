---
title: CMesS
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 31
---

# CMesS

<img width="968" height="336" alt="image" src="https://github.com/user-attachments/assets/3571707a-986e-4a94-8ae1-dd508c40c6c7" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Can you root this Gila CMS box?

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 22, and 80```. 

Opening the webpage, we see a blog page using Gila CMS.

I tried finding the version number by perusing the webpage but no luck.

---

I ran cmseek to see if I could find out the version number that way, but cmseek freaked out and couldn't identify the CMS at all.

I tried gobuster, but tehy must ahve set up the webpage in a defensive way because every gobuster request recieved a 200 response.

Giving up on gobuster, I just tried out subdirectories that I thought would make sense for a CMS like ```/login```. Which exited. To no one's suprise.

But this login page had nothing of interest either, so back to the drawing board.

---

I decided to try and fuzz for subdomains, but they had the same defensive techique that we encountered with gobuster.

I didn't want to give up on fuzzing as easily as I did subdir scanning, so I googled for a way to bypass this adn found that I can filter out the responses of the dummypages based on their wordcount since they were all the same.

Doing this, I found the subdomain ```dev```.

### Visiting the subdomain

Upon visiting the subdomain, it was a chat between andre@cmess.thm and support where he asked if they fixed a bug and also asked to change his password. The password waws there in plaintext. Easy easy easy.

### Gila CMS Admin Panel

Using my new found credentials, I logged into the Admin panel for Gila CMS and found a version number: 1.10.9.

With hope in my heart, I used searchsploit to try and find an exploit for Gila CMS v1.10.9, and I hit the jackpot. 

An exploit for authenticated rce with the version being on the dot.

This exploit just uplaoded a reverse shell to the website using cookies and the php session id. As simple as that.

Executing the script, I got a callback, cool and awesome.

<img width="824" height="594" alt="image" src="https://github.com/user-attachments/assets/072258dd-739e-46d8-bea5-ffd95220dc0b" />

## Shell enum as www-data

Enumerating the box as www-data, I found a mysql database and teh credentials for it in the config.php file.

I logged into the DB and found andre's password hash and set up hashcat in the background to try and crack it.

I also found a backup wildcard cronjob ran as root, but i needed access to andre's directory to exploit it so I need to escalate to his account first.

Doing further enumeration in different directories, I found a backup apssword file in plaintext in the ```/opt``` directory, and what do you know it works. No hashcat needed...

## Valid Andre SSH Session/priv esc

Now I can exploit the cronjob.

<img width="1891" height="365" alt="image" src="https://github.com/user-attachments/assets/bfae2c2c-807e-41cf-adfe-397eabb258a0" />

Bam. Pro stuff.

But what I did was make 3 files: 1 mkfifo shell file, 1 file that would act like the --checkpoint=1 flag so tar would take it as a flag and not a file, and tehn finally a similar file that acted as a flag to tar that told it on checkpoint 1 use shell to execute this script.

Feels good to be root.


---

## Solution Steps

1. Use ffuf to find dev subdomain
2. login with credentials found on that subdomain
3. use rce exploit on the gila cms version found
4. find andre's password backup in ```/opt```
5. ssh as andre
6. exploit wildcard crontab


## Thoughts

I really enjoyed this box becauase it felt good to see my practice payoff and that I was able to privesc to root without needing to refer to a writeup. I struggled a bit though with andre's password, though the lesson learned is never just enumerate with ls always do ls -la. Highly reccomend this box.

---

Thank you for reading!

