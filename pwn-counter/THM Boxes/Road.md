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

I decided to take a look at the ports that linpeas output and realized that there were default ports for mysql and monogDB.

#### Enumerating MongoDB

I tried opening up a mongo shell to see if teh DB was misconfigured to allow passwordless access, and to my luck it was.

I enumerated around trying to find any useful databases and tables and I found a valid set of credentialsa for the webdeveloper user: ```webdeveloper:BahamasChapp123!@#```.

### SSH Session Enum

Using the valid credentials, I was able to get a valid SSH Session as the webdeveloper user.

I checked for SUID binaries again to see if I had access to any new file directories with SUID binaries, but none were found :(.

---

Then, I did sudo -l to try and see if I can run sudo, and I was able to run a custom backup compiled program with sudo as root.

I tried to find a way to exploit this binary by maybe possibly putting something in the tar file directory so it would grab that and somehow give me a shell, but I couldn't find anything when google searching how to execute my idea, so I pocketed the idea and looked for another vector.

---

I then noticed that I had an unusual permission on my user from the sudoers file. LD_PRELOAD was included under env keep. This is interesting because usually, when running SUDO, certain environment variables are wiped to prevent potential privilege escalation, and with this permission all are wiped but this env variable. Interesting.

After googling how to exploit this, I cam across this amazing [video](https://www.youtube.com/watch?v=bzjnIi5u9OQ&t=308s) by Conda. 

Essentailly, the ```LD_PRELOAD``` environemnt variable is a way to tell the linker of a program to laod a shared library before the program boots up. And since shared libraries are compiled and executed by the linker and compiler before a program boots up. 

So, if we have a malicious shared library that just sets our uid and guid to 0 (root) and spawn /bin/bash, we can get a root shell before the porgram even boots up.

This is essentially the C version of the python mopdule hijacking box I did a while ago.

## priv esc

So, by compiling the following code into a shared library using this command: ```gcc -fPIC -shared -nostartfiles -o lol.so lol.c``` 

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
unsetenv(“LD_PRELOAD”);
setgid(0);
setuid(0);
system(“/bin/bash”);
  }
```

Then by setting the LD_PRELOAD variable and running the binary with sudo (```sudo LD_PRELOAD=/tmp/pe.so /usr/bin/sky_backup_utility```), we get a root shell.

<img width="1281" height="716" alt="image" src="https://github.com/user-attachments/assets/113d07b7-4380-4b67-bae9-f5ec1c5d9b7e" />


---

## Solution Steps

1. Create account in login portal
2. Reset Admin password
3. Upload php reverse shell as profile picture
4. navigate to shell in ```/profilepictures```
5. Escalate to ```webdeveloper``` user by finding their creds in the Mongo DB
6. Exploit ```LD_PRELOAD``` not being wiped to laod malicious library that spawns root shell.


## Thoughts

I really enjoyed this box becuase I had always ignored the permissions that were found for the user in the sudoers file, I really just focused on whether or not I can run a binary as sudo. This box sharpened my enumeration skills and also made me realize I need more practice in noticing the difference between default things found on a machine vs unusual things found on a machine. Highly reccomend.

---

Thank you for reading!

