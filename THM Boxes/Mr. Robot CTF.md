---
title: Mr. Robot
parent: THM Boxes
nav_order: 1
---

# Mr. Robot CTF

<img width="1009" height="286" alt="Screenshot 2026-06-28 142144" src="https://github.com/user-attachments/assets/b1b0a0eb-0357-4813-93da-f2d96bd0d398" />

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Challenge Description

> Placeholder description — replace with the real challenge prompt once you have a real UAF writeup to add here.

## Recon

For the recon phase I started out as always with an nmap scan using ```nmap -sC -sV -vv target_ip```. This scan revealed 3 ports open: '''port 80, port 443, and port 22'''. 

Seeing port 80 open, my first instinct is to check what was being served on that port via a browser. It was just some Mr. Robot references and some cool videos.

I tried looking at the source code for the custom CLI animation it served up upon loading the webpage, but all I found was a hidden 420 command which served a bob marley quote, very nice. 

I was a bit stuck so I checked a writeup and couldn't believe I ahd forgotten one of the most easy enumeration steps: gobuster.

I ran ```gobuster dir ip -w big.txt``` and found a robots.txt file. Perfect.

I also found a login page and found out the content was being served up via wordpress.

I tried the classic ```admin:admin, admin:password, user:password``` but to no luck, but all is not lost becuase I got an error indicating that the login site was telling me whether or not my username was valid.

Navigating to the robots.txt directory, I found the first flag and some kind of Mr. Robot wordlist.

I tried thinking of how to use this wordlist. It was a strange wordlist. It was about ~850,000 words that looked like a mix of usernames and potentially passwords.

I was stuck as I was not sure how to find a valid user and learned from peeking at the writeup that you can not only use hydra for brute forcing a password, but also for bruteforcing a username, smart.



So, I used the weird worlist to try and bruteforce a username out of the word press site with the command ```hydra -L weird_wordlist.txt -p test http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&rememberme=forever&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.64.166.77%2Fwp-admin%2F&testcookie=1:Invalid username." -V -t 64 -f```.

With this command, I found a valid username of Elliot, nice.

My first instinct with this valid username was to run through the rockyou wordlist but I was not getting any hits and then realized that the weird wordlist also had passwords so I ran ```hydra -l Elliot -P weird_wrordlist.txt http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&rememberme=forever&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.64.166.77%2Fwp-admin%2F&testcookie=1:Invalid username." -V -t 64 -f```.

After a literal enternity becuase the password was like number 820,000, I found a valid password for the user Elliot and was able to login ╰(*°▽°*)╯

Perusing around the configuration settings I found an area to upload images and of course had to try uploading the classic php rev shell but there were security restrictions on place on file extensions.

I thought about encoding a reverse shell in an image, but I remebered not only an easier but faster way.

In one of my previous boxes 




## Exploitation

Walk through the heap layout, the freed chunk reuse, and how control flow or data was hijacked.

## Solution Steps

1. Step one
2. Step two
3. Step three

## Flag
