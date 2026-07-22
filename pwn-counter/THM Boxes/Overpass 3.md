---
title: Overpass 3
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 29
---

# Overpass 3



## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> You know them, you love them, your favourite group of broke computer science students have another business venture! Show them that they probably should hire someone for security...

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 21, 22, and 80```. 

Visiting the webpage, we see a webpage for a hosting solution.

I try checking the source code for any hidden info, but nothing.

I ran cmseek and nuclei to no avail, but my gobuster scan revealed one subdirectory: ```/backups```.

Going into the the ```/backups``` directory, I see it is an exposed file listing with a ```backups.zip```.

Downloading it to my machine, I unzip and get a gpg encrypted spreadsheet and a private key.

Right off the bat  I tried to see of this private key would be able to decrypt the spreadsheet.

I used ```gpg``` to import the key into my keyring, directed the output of the decrypted file content to another file, and opened it with libreoffice since it was a spreadsheet.

Here I see a list of usernames and passwords along with credit card details.

<img width="674" height="78" alt="image" src="https://github.com/user-attachments/assets/fb5f25f0-45cf-42a0-93cb-02e82e6b71ec" />

### Checking for password reuse

With 4 sets of credentials, I start checking for password reuse.

I start with ```paradox:ShibesAreGreat123```.

I tried them with ssh but didnt get a session established, but got one with ftp, nice.

### Inside FTP

I looked around for any useful info and found a .jpg file named ```hallway.jpg```. I tried checking for steganography using ```steghide and stegseek```, but I think that was a dead end.

I then realized that the ftp root directory's contetns were exactly what was on teh webpage, we essentially had a shell with only file download and upload capabilities.

### Initial Foothold

Knowing this, I created a new directiory ```a``` and uploaded my php reverse shell. 

Navigating to the shell in the webpage, we get a callback on my listener, nice.

<img width="900" height="179" alt="image" src="https://github.com/user-attachments/assets/ed45ce82-470c-4c45-9588-daf18dbbbf0a" />


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

