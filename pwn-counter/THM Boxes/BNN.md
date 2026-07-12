---
title: Brooklynn Nine Nine
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 23
---

# Brooklyn Nine Nine

<img width="943" height="272" alt="image" src="https://github.com/user-attachments/assets/b0969614-4342-4308-9f16-f5c1957a8321" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> This room is aimed for beginner level hackers but anyone can try to hack this box. There are two main intended ways to root the box.

## Recon

As always, I satrted off with an nmap scan using ```nmap -sC -sV -vv ip``` which revealed services on ```ports 21, 22, and 80```.

Visiting the ftp server, ther is a note telling jake to change his password, so I thought to do an ssh brute force on the user Jake with teh rockyou wordlist, but that turned out to be a dead end.

Visiting the webpage, there is an image of the show's logo, and going to check the source code there is a comment that syas "Ever heard of steganography?"

I downloaded the image and tried to use steghide to extract its contents, but it was password protected.

To bypass this, I used stegseek with rockyou.txt and blew through the worlist in a literal second and I got the extracted info which was Holt's password of ```fluffydog12@ninenine```.

I tried to ssh using Holt, but didnt get in.

I tried holt lowercase and the password and I got in.
  creds: ```holt:fluffydog12@ninenine```

## Priv Esc

Looking for SUID binaries I was at a dead end, but when i checked my sudo permissions, I was able to run nano as root with no password.

I checked GTFObins and perfect there was an entry for a privileged shell using nano with sudo.

I ran an nano, did ctrl+R and ctrl+X and then typed ```reset; sh 1>&0 2>&0``` and got a root shell within nano, nice.

<img width="522" height="182" alt="image" src="https://github.com/user-attachments/assets/6ce04bbc-9841-49bd-821e-3fc28aab6f6b" />


## Solution Steps

1. use stegseek with rockyou on image found on webserver.
2. use creds to ssh into target box
3. run nano then ctrl+r and ctrl+x then for the command write ```reset; sh 1>&0 2>&0```.

## Thoughts

This was a super duper easy box and I didnt feel like I learned anything or sharpened my skills, but it was fun regardless.

--

Thank you for reading!

