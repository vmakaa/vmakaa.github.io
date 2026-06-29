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

## Box Description

> A Mr. Robot Themed capture the flag.

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

In one of my previous boxes, I had learned that since wordpress serves up php pages, I can just replace one of the pages with the reverse shell code and tehn navigate to it. I chose the 404 page as I ahd a little trouble with my usual index page, naviagted there, and got a callback on my listener with a shell (￣_,￣ )

As always, I upgraded my connection using the command ```python -c 'import pty; pty.spawn("/bin/bash")' ```. It really just is cosmetic but I find it helps to see at all times what directory im in.

Looking around I found the second flag but couldn't read it, uh oh.

I realized I was still the daemon user but I found a conveniently placed file in teh same directory containing the md5 hash of the password of the user robot.

I first tried to crack the hash using crackstation.com and got the password of ```abcdefghijklmnopqrstuvwxyz```.

I remebered ssh was on the machine, so with the credentials ```robot:abcdefghijklmnopqrstuvwxyz``` I tried to ssh into the machine and I got a successfull ssh connection o(*￣︶￣*)o

I tried to enumerate what sudo prvileges I had using ```sudo -l``` but found I was not able to run sudo at all (╬▔皿▔)╯

Something new that I have been trying to drill into my post-foothold enumeration routine is to look at all the files with suid bits using the command ```find / /perm +6000 2>/dev/null``` (thank you writeup).

I was looking trhough all the suid binaries, cross referencing them with GTFObins to see if they had a section exploiting it with SUID and found that nmap had the SUID bit set which leads me to my exploitation phase.


## Exploitation

According to GTFObins, when nmap has teh suid bit set you can run an namp interactive session and then spawn a root shell. So I followed the steps GTFObins listed. I

I ran ```namp --interactive``` which brought me to an nmap shell that looked like this ```nmap>```.

Then I did ```nmap>/bin/sh``` and was greeeted with the lovely ```#``` shell prompt in which I navigated to the root directory adn got the final flag, adding another to my pwn counter.

## Solution Steps

1. Find valid username via hydra and valid password via hydra using the wordlist found via robots.txt
2. Login to the wordpress site and replcace 404.php with php reverse shell code with a listener set up
3. Exploit the nmap binary that ahs the SUID bit set to get a root shell

## Thoughts

This box was rated as a medium difficulty on THM, but I think it should have been rated easy. I am pretty proud that I was able to root the box by only having to refer to the writeup 3 times, which is still a lot but compared to how often I had to when I first started playing CTFs is a gerat improvement which Im happy with. 

I alwasy find it fun and interesting to exploit SUID bit binaries so that was pretty fun, and overall I enjoyed the box.

--

Thank you for reading!

