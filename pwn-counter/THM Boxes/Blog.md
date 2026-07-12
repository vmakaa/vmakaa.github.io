---
title: dogcat
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 22
---

# Blog

<img width="918" height="244" alt="image" src="https://github.com/user-attachments/assets/38d26c5e-8a7d-4db5-8759-40b9301c1550" />

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Billy Joel made a Wordpress blog! 

## Recon

Since this is a wordpress based box, it is safe to assume that the site is being served up on port 80. 

So first I decided to do ```cmseek -u http://box_ip``` and found 2 valid usernames: ```bjoel``` and ```kwheel```.

<img width="761" height="417" alt="image" src="https://github.com/user-attachments/assets/bbc48032-c59b-46ee-b48b-115c0801c239" />

After running ```gobuster dir -u http://10.64.152.178:80/ -w /usr/share/wordlists/dirb/small.txt -r```, four subdirs are found.

<img width="433" height="126" alt="image" src="https://github.com/user-attachments/assets/2397197e-de6e-4b9a-b3b1-73a6c133c478" />

After going to the ```/admin``` subdir, I am presented with a admin login page.

With my 2 valid users I run hydra for both usernames using ```hydra -l bjoel -P /usr/share/wordlists/rockyou.txt -t 64 -vV -f -I 10.64.152.178 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password"``` and ```hydra -l kwheel -P /usr/share/wordlists/rockyou.txt -t 64 -vV -f -I 10.64.152.178 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password"```.

From the kwheel scan, I find a valid password giving me my first set of creds ```kwheel:cutiepie1```.

<img width="652" height="26" alt="image" src="https://github.com/user-attachments/assets/13aa498c-cff1-4cc6-8770-84f457dc0764" />

After logging in and messing around a little, I couldn't find exploitable areas so I searched online for vulnerabilities related to wordpress 5.0.0.

I found an image crop vulnerability that led to RCE but I was having a ton of trouble with getting the 3 different manual PoCs I tried to work, even with using Claude, so I used msfconsole. I try not to use metasploit as Im aiming to take the OSCP, but I have spent close to an hour trying to manually exploit this insane, niche vulerability with no luck, so metasploit it is.

I found the msfconsole module for this exploit, loaded it, set the options, executed it, and got a shell, finally.

Looking around for suid binaries, I found an unusual file at ```/usr/sbin/checker```. I instictively tried to cat it out but it was a bunch of garbled nonsense which led me to realize it was a binary. 

I was trying to rely on metasploit as little as possible and got the idea from a writeup to use ltrace for the binary.

So, I executed the binary and its just a simple program checking if the admin environemnt variable exists.

<img width="463" height="88" alt="image" src="https://github.com/user-attachments/assets/0e74c212-0f80-47f8-9693-fa508ab9d5e7" />


### priv esc

With this knowledge I did ```export admin="admin"```, ran the checker binary, and my uid was set to root, nice.

Its strange that I was able to get root.txt before user.txt, but thats because the user.txt was behind a directory that had higher permissions than what ```www-data``` had so I was able to do a find command for user.txt once I had root and find it.

<img width="519" height="277" alt="image" src="https://github.com/user-attachments/assets/47c470e2-10f1-44c0-a97b-06e9f95fa891" />

---

## Solution Steps

1. CMSeek to find valid usernames
2. Use hydra with kwheel username and rockyou.txt to find kwheel:cutiepie1
3. Use metasploit with the crop image exploit to get a shell
4. find the suid checker binary and use ltrace to understadn what it does
5. do export admin="admin" (or anything) and get root

## Thoughts
This box was very very interesting. Its a good box, I just was really frustrated that I couldn't get the manual scripts to work and had to rely on metasploit (even the searchsploit exploit required metasploit???). I thought the privellege escalation was very creative and it was fun to figure out, though I wish I had remembered about the ```ltrace``` command as I have used it before and I could have referenced the writeup less. Overall, this box was fun and taught me how to analyze a binary that I dont have download access to.

---

Thank you for reading!

