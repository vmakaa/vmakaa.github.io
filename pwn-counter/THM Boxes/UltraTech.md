---
title: UltraTech
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 23
---

# UltraTech

<img width="944" height="244" alt="image" src="https://github.com/user-attachments/assets/bb0bedde-9d21-496a-87b4-3c97e1556eac" />

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> The basics of Penetration Testing, Enumeration, Privilege Escalation and WebApp testing

## Recon

As always I start off with an nmap scan which revealed that there were services on ports ```22, 8081, 31331```.

The first port I decided to test was ```port 31331```. Upon entering the correct URL for that port in firefox, I am greeted with UltraTech's webpage.

<img width="1478" height="657" alt="image" src="https://github.com/user-attachments/assets/fcd2a517-9db0-4ac5-9109-c75dfb41e398" />

Running ```gobuster dir http://box_ip:31331 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -t 100``` I find some of standard subdirs for a webpage like ```/images and /js```.

In the ```/js``` subdir, I find the source js code which uses the API hosted on port 8081.


```javascript
(function() {
    console.warn('Debugging ::');

    function getAPIURL() {
	return `${window.location.hostname}:8081`
    }
    
    function checkAPIStatus() {
	const req = new XMLHttpRequest();
	try {
	    const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
	    req.open('GET', url, true);
	    req.onload = function (e) {
		if (req.readyState === 4) {
		    if (req.status === 200) {
			console.log('The api seems to be running')
		    } else {
			console.error(req.statusText);
		    }
		}
	    };
	    req.onerror = function (e) {
		console.error(xhr.statusText);
	    };
	    req.send(null);
	}
	catch (e) {
	    console.error(e)
	    console.log('API Error');
	}
    }
    checkAPIStatus()
    const interval = setInterval(checkAPIStatus, 10000);
    const form = document.querySelector('form')
    form.action = `http://${getAPIURL()}/auth`;
    
})();
```

From this source code, we can see two paths the API uses: ```/ping and /auth```.

Checking ```robots.txt```, I found the subdir ```/partners``` which had some form of login field.

Using the browser tools, I viewed the web requests this page made, and it was using the ```auth``` functionality of the API to validate the credentials, and then it was using the ```ping``` functionality to test the connection to the host box.
> writeup helped me realize this

<img width="1915" height="85" alt="image" src="https://github.com/user-attachments/assets/e8415ad2-8ed3-432b-9db4-da05a76d2831" />

It looked like that every 5 seconds, a ping command would be sent to the host box to test connectivity.

I opened the ping command in a URL to see if I could test ```8.8.8.8``` to see if the IP was hardcoded to the API functionality and: 

<img width="1006" height="106" alt="image" src="https://github.com/user-attachments/assets/683c716c-f923-4d56-b0a0-db0a8bfc71d1" />

Success! A new pathway had opened: command injection.

Using [Hackviser's command injection cheatsheet](https://hackviser.com/tactics/pentesting/web/command-injection), I tested a couple of payloads to replace the ip for.

After testing a couple I found that the backticks worked as when I did ping?ip=`whoami` (cool bts: it works becuase the backticks evaluate whatevers inside of them ebfore executing the rest of the cmd line).

<img width="741" height="108" alt="image" src="https://github.com/user-attachments/assets/bb8d4eb5-6d54-4f5c-b370-ba68b3a63359" />

With this knowledge, I tried ls to see what was in my directory and what a stroke of luck, a database right there. 

<img width="714" height="107" alt="image" src="https://github.com/user-attachments/assets/60ab84da-9d5d-4f64-9e06-f13dbe8dd5ed" />

I tried to cat it out and got some jarbled output, but was able to determine the usernames and where their hashes began and ended.

<img width="1025" height="37" alt="image" src="https://github.com/user-attachments/assets/acdfb5c2-15e1-4f46-8bb1-ef4ed4ccb27d" />

Cracking these hashes, the new set of creds I had were ```r00t:n100906 and admin:mrsheafy```, nice.

I tried to login to the partners page with both credentials, but both had the same result, a note left by user ```lp1``` which told me to pivot where I was looking.

<img width="523" height="207" alt="image" src="https://github.com/user-attachments/assets/b792c79a-10d3-43f2-b378-f52d9e064bc7" />

Trying both credentials via ssh, I got an ssh session with the creds ```r00t:n100906```.

Looking around I tried looking for SUID binaries but to no luck and my user was not able to run sudo at all.

Thats when I ran ```id``` and saw that my user was in the dokcer group.

<img width="515" height="30" alt="image" src="https://github.com/user-attachments/assets/a73fd382-681f-4276-b9b6-e7a489c48567" />

### priv esc

On gtfobins, I discovered that docker was an exploitable binary, and no matter what privileges you had, you can escalate your privileges to root as long as you were in the docker group or the docker user.
> found thanks to writeup, didnt know docker could be used as a priv esc vector

Running the command ```docker run -v /:/mnt --rm -it <docker_image> chroot /mnt /bin/sh``` as any user whether privileged or not would escalate your privileges.

Using ```docker ps -a``` I found 3 images named bash, so I used the command ```docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh``` and got root privileges.

<img width="976" height="80" alt="image" src="https://github.com/user-attachments/assets/7b610bf0-a20b-4b5a-be2a-df0852d88b78" />
<img width="957" height="71" alt="image" src="https://github.com/user-attachments/assets/d822fb80-0ba5-471d-9286-1a65b800bf0d" />

Looking around, I found a private.txt but it was just some kind of autobiography 😭

<img width="584" height="179" alt="image" src="https://github.com/user-attachments/assets/b5dfd5f3-1277-4d94-928f-d950efd0ef16" />

The CTF itself was looking for the first 9 characters of the root user's private ssh key, so I ran the command ```find / -name *id_rsa 2>/dev/null | grep rsa``` and found the root user's private ssh key.

<img width="370" height="53" alt="image" src="https://github.com/user-attachments/assets/f070c68e-8d74-4567-9edb-43cd84c49682" />
<img width="541" height="95" alt="image" src="https://github.com/user-attachments/assets/bc66cb2d-9da7-49d7-8722-ede2b3f9015f" />

---

## Solution Steps

1. Find out how the webpage uses teh API
2. Command inject the ```?ip=``` parameter in the ping commadn for the API using backticks
3. Using that, get the usernames and their hashes.
4. Use those creds to ssh into box
5. priv esc via docker


## Thoughts
This was another erally fun box. I thought the priv esc vector was really novel and fun. I also liked the essence of realism associate with it and this was my first time doing a box with an API so that was really fun. Overall 10/10 box, I really enjoyed it.

---

Thank you for reading!

