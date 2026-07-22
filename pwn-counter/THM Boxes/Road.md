---
title: Road
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 30
---

# Road

<img width="946" height="273" alt="image" src="https://github.com/user-attachments/assets/6a0c60c0-142c-4465-8189-9bcab1a9e7d6" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> [CTF] Inspired by a real-world pentesting engagement

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 22, and 80```. 

Opening the webpage, we see a website for an E-Commerce solution.

Enumerating around. I find a login page.

I try the low hanging fruit and do ```admin:admin``` and ```admin:password``` but to no avail so that was a dead end.

I found that it lets you register a user sp I registered ```root@root.com``` with a password of ```hello``` and started enumerating the user's dashboard.

At first, it didn't seem like there was much to the user dashboard.

There was a password reset option and the ability to add information about yourself, but when I scrolled towards the bottom to edit profile, I noticed that changing your profile picture was locked with a note saying if you would like to do so, please contact ```admin@sky.thm```.

So now we have a valid admin user.

### Resetting Admin Password

At first, my first instinct was to hydra the admin account for a valid password, but since we have a reset password functionality, I decided to test how that worked under the hood.

I opened up burpsuite and captured the request, and the reset password functionality was just cleartext credentials being sent to what I presume was a php function. 

I tried setting the email to ```admin@sky.thm``` and the password to ```hello``` and I got a response saying the password reset was successful!

<img width="1191" height="575" alt="image" src="https://github.com/user-attachments/assets/c317b322-4c92-4f84-81ce-b124c5342ab2" />

Let's see if it really worked.

<img width="1906" height="697" alt="image" src="https://github.com/user-attachments/assets/d02593c3-9195-4b58-a9bc-c3eb6c03512a" />

Nice! Let's do some more enumeration.

### Getting a shell through the edit profile picture functionality

My first instinct was to try and upload a php reverse shell as the profile picture, but I could not tell if teh upload was successful or not.

I tried looking around as to where the folder holding all the profile pictures would be.

Maybe the assets directory i found from my initial gobuster scan? Not there unfortunately.

I checked the html code of the profile.php page and there was a commented out line with the subdirectory ```/v2/profileimages```. So this is where it is.

To test if this really was where it was stored since a php reverse shell just buffers, i uploaded a random file as the profile picture and sure enough I was able to navigate to it, lets get a shell.

<img width="1145" height="738" alt="image" src="https://github.com/user-attachments/assets/b9025360-c2c6-47a4-a050-d9ce22052808" />

Nice.

### Shell enumeration

Enumerating the shell, we check for sudo but dont have the password, check for suid binaries but none are exploitable, check crontab but no cronjobs. 

I decided to run linpeas.sh and went down the rabbithole of pam_cap.so but that led to nowhere.

I decided to take a look at the ports that linpeas output and realized that there were default ports for mysql and monogDB

### priv esc

<img width="1281" height="716" alt="image" src="https://github.com/user-attachments/assets/113d07b7-4380-4b67-bae9-f5ec1c5d9b7e" />


---

## Solution Steps

1. Find the ```/backups``` directory and download ```backups.zip```


## Thoughts
I really enjoyed the priv esc technique this box featured. Prior, I wasa not familiar with nfs drives and exploiting them. But, now I feel if I see it again I will be prepared, and I always appreciate when a box teaches me a new, useful skill in preparation for the OSCP. I though the initial foothold was pretty creative too, it took me a while to realize that the ftp share directory was teh web directory and it seemed so obvious once I realized. Overall a great and fun box. Definitly reccomend.

---

Thank you for reading!

