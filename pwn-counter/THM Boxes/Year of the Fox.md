---
title: Year of the Fox
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 34
has_children: true
---

# Year of the Fox


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Be sure to check out the [Formal Report(https://vmakaa.github.io/pwn-counter/THM%20Boxes/Formal%20Report%20(OMWS).html) once you are done reading!

---

## Box Description

> Don't underestimate the sly old fox...

## Difficulty

> Hard

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 139, 80, and 445```.

---

### SMB Enum

Using SMBMAP, we find that there are 2 shares that are inaccessible without the proper credentials.

<img width="1097" height="444" alt="image" src="https://github.com/user-attachments/assets/64ff8264-d2ed-45af-9729-5bfefcb0d7ff" />

---

Then using enum4linux, we find two local users.

<img width="619" height="95" alt="image" src="https://github.com/user-attachments/assets/163bb00f-2100-4200-a851-80c037254c13" />

And thats all the useful info I found from enumerating SMB.

---

### Web Enum

Visiting port 80, there is a prompt to enter a username and password. If not provided, a 401 unauthorized error is returned.

---

I tried to do hydra for the users ```fox``` and ```rascal```, and found a password for rascal for the webserver authentication ```rascal:washburn``` (note that the creator said the password changes upon reboot of the machine).

---

Authenticating in, access is granted to a website that looks to be a custom search engine. 

<img width="1267" height="605" alt="image" src="https://github.com/user-attachments/assets/b56c32f6-0b44-4454-a8ff-a87aa08003ae" />

---

Trying the low hanging fruit of ```/etc/passwd```, it is apparent that the backend is filtering out special characters such as ```/``` from even being typed.

To get around this I captured the request in burp, and did my work via repeater.

<img width="1092" height="373" alt="image" src="https://github.com/user-attachments/assets/777dbb8d-9323-48d9-a595-575199b71816" />

It looks like that the search engine tries to find a specific set of files that have the letters provided in the target parameter.

Command Injection seems to be the most likely vulnerability here.

---

#### Command Injection

When performing command injection, you are usually doing so via the URL, as it has the search functionality implemented via ```?s=```.

This time, there is a custom search functionality using a parameter in a JSON sent to the webserver.

---

Noticing the quotation marks, the question arises whether or not your input is passed to the application with quotation marks still surrounding them, so I looked for a way to break out of the quotation marks.

Looking online and asking some LLMs, I found an interesting trick.

If you think about it, since this is a php functionality, the underlying backend most likely does something like this:

```system("cmd to run " . $_GET['target']);```

But, if it wraps your value in double quotes, a traditional injection does work and looks like this to the underlying system ```"; id"```.

So in order to bypass those wrapping quotes, you need to supply your own quotes so it then breaks out the quotes and looks like this in the CLI:

```cmd to run "" ; id ""```

Thereby, escaping the quotes and running your command.

---

Since this is a custom search engine functionality, I think the php app just checks an array and returns an array on array items that match a letter you sent. So instead of the traditional whoami, lets use the ```sleep``` command to verify RCE.

if the following is supplied in the JSON:

```JSON
{
  "target": "\"; sleep 10 #"
}
```

(Since JSON requires your input to be in double quotes to pass to the server, you need to use ```\"``` to tell JSON to interpret the ```"``` literally so you can perform the double quote escape and the ```#``` at the end is a comment in php rendering everything after your command non-executable)

The server takes 10 seconds to respond, verifying RCE.

---

### RCE to Reverse Shell

Knowing we now have verified RCE, its time to upload a reverse shell.

Now I can't just put the reverse shell after the semi-colon because there is nothing to execute it, so I echoed the command and then piped it over to bash but I got an invalid character error.

To bypass this, the reverse shell payload was base64 encoded and then piped over to ```base64 -d``` and then piped over to bash and the listener on the attacking machine received a callback.

<img width="567" height="234" alt="image" src="https://github.com/user-attachments/assets/91db7c4e-907c-4731-bcde-499ac2d5e02b" />


Command used:

<img width="548" height="426" alt="image" src="https://github.com/user-attachments/assets/08a0eae0-60ea-4ac4-9294-4410aaf3c4a1" />

#### Checking out search.php

Out of curiosity, I wanted to see if my theory was correct.

I was wildly wrong. Turns out we were in an exec command using find. But our quotation escape was spot on because as it turns out, our input was wrapped in quotations.

---

### Shell Foothold

Now that I have an initial shell, its time to enumerate.

But before enumerating, I wanted to get socat onto the box and establish a socat connection because the connection using the burp payload was not stable.

---

I took a socat static binary, uploaded it to the box, and established a socat TTY connection.

Much better.

---

Enumerating around, I found a hash for pascal, but cracking it showed me that it was the same one as before. I thought it might've been for something more juicy. Oh well.

I took note of a bunch of privesc vectors I found in linpeas. I found a potential writable path abuse and a potential sudo abuse so I noted those down for later.

---

Looking around in the linpeas output, I noticed that ssh was bound to the localhost. I tried using ssh as ```www-data```, but I did not have perms to do so.

I had the idea to port use socat to bind ssh to the box's ipv4 address on port 2222, so I did the command ```./socat TCP-LISTEN:2222,fork,bind=10.65.183.55 TCP:127.0.0.1:22``` and no error messages. Looking good so far...

On my attack box, I tried sshing into the box using its ipv4 address and using port 2222, and success! I got the ssh prompt.

I tested for password reuse for teh rascal user, but no luck.

I tried brute forcing the user ```fox```'s ssh creds with hydra using the command ```hydra -l fox -P /usr/share/wordlists/rockyou.txt ssh://10.65.183.55 -s 2222 -vvv -t 64``` and I got a password!!

<img width="527" height="21" alt="image" src="https://github.com/user-attachments/assets/6c428e09-f166-411e-ac2a-3b2479ee91a1" />

I tried the password and it worked. Hallelujah.

<img width="639" height="319" alt="image" src="https://github.com/user-attachments/assets/7e68d535-eb89-4732-aeb1-d1e9df7ce5c2" />

### SSH Session Foothold

Getting an ssh session, I checked my sudo perms and I was able to run a binary called ```shutdown``` as root. Pulling the binary onto my machine and loading it into ghidra, its safe to say I got tricked by the red herring....

<img width="280" height="141" alt="image" src="https://github.com/user-attachments/assets/95314e9d-e113-49a5-a99d-2dca810d302a" />

---

With that going nowhere, I decided to try one of the linpeas privescs I found. 

I tried the sudo exploit, but it required me to use the make command which wasn't on the box nor was gcc and the box didn't have any internet connection.

So that left me with the writable path exploit left to try.

---

To be cont

## Solution Steps

1. Use nuceli to find the rce vulnerability on the web server
2. get a shell and escalate to root uid using python set uid capabilities
3. break out the docker container using OMI on port 5986 exploit to get reverse shell as machine host


## Thoughts

This was a really fun box. Even though the initial priv esc was really easy, I found breaking out the docker container to be a niche and cool path to learn about and add to my toolbelt. Definitely Recommend. 

---

Thank you for reading!

