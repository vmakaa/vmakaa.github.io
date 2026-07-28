---
title: Bookstore
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 32
---

# CMesS



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

I checked the current UID and GUID by reading ```/proc/self/status``` and then cross referencing those values in ```/etc/passwd``` and found the user ```sid``` was running the flask app. Found the full path of the flask app which was ```/usr/lib/python3/dist-packages/flask/app.py```. Found the MAC Address by reading the active network interface from ```/proc/net/arp``` and then reading ```sys/class/net/ens5/address``` to get the MAC address of ```0a:ff:f1:73:c4:eb ```.








---

## Solution Steps

1. Use ffuf to find dev subdomain



## Thoughts

I really enjoyed this box becauase it felt good to see my practice payoff and that I was able to privesc to root without needing to refer to a writeup. I struggled a bit though with andre's password, though the lesson learned is never just enumerate with ls always do ls -la. Highly reccomend this box.

---

Thank you for reading!

