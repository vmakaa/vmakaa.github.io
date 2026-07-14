---
title: Boiler
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 24
---

# Boiler

<img width="912" height="251" alt="image" src="https://github.com/user-attachments/assets/b129a7f8-802a-45e4-b410-d57f87090a4a" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Intermediate level CTF. Just enumerate, you'll get there.

## Recon

As always, I started with an nmap scan which revealed services on ```ports 21, 80, 10000```.

<img width="584" height="489" alt="image" src="https://github.com/user-attachments/assets/b9cb112a-97d9-41fa-bbec-56242be0c190" />

Looking at the fto scan, I noticed that Anonymous login was allowed, so I decided to see what I could find.

Ftping into the servere and executing ```ls -la``` we find a hidden file named info.txt

Catting it out, we get the following: ```Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!```.

I could immediately tell it was a ROT13 cipher, so using cyberchef to decode it we get the follwoing message: ```Just wanted to see if you find it. Lol. Remember: Enumeration is the key!```

(╬▔皿▔)╯

I decided to do another scan on all ports, and I found an additional port open ```port 55007``` that was just ssh on a non-standard port.

I decided to do another gobuster scan on ```http://box_ip/``` and found a joomla page. 

After using ```cmseek``` on the joomla url, I couldn't find anyhting, but noticed that ```cmseek``` found a few subdirs under ```/joomla/administrator``` which inspired me to do a gobuster scan on ```http://box_ip/joomla``` which ahd a ton of output.

<img width="418" height="373" alt="image" src="https://github.com/user-attachments/assets/d88b644a-9950-47b9-9b17-79279c00576f" />

The most notable find was the ```/test``` subdir which led me to a ton of files.

<img width="535" height="576" alt="image" src="https://github.com/user-attachments/assets/e81ef551-2217-47c2-9526-1b3655d14801" />

In the docker_compose file under jenkins, I now had the credentials to their database.

```yml
version: '2'

services:
  test:
    image: joomlaprojects/docker-${PHPVERSION}
    volumes:
     - ../..:/opt/src
    working_dir: /opt/src
    depends_on:
     - mysql
     - memcached
     - redis
     - postgres

  mysql:
   image: mysql:5.7
   restart: always
   environment:
     MYSQL_DATABASE: joomla_ut
     MYSQL_USER: joomla_ut
     MYSQL_PASSWORD: joomla_ut
     MYSQL_ROOT_PASSWORD: joomla_ut

  memcached:
    image: memcached

  redis:
    image: redis

  postgres:
    image: postgres
```
Enumerating further using common.txt on gobuster, I find more subdirs that werent previously found. 

<img width="446" height="233" alt="image" src="https://github.com/user-attachments/assets/bd292a84-9a90-498b-bb3d-042096a4f9b3" />

Looking around, I find that the subdir ```/_test``` is a tool titled ```sar2html```.

<img width="984" height="579" alt="image" src="https://github.com/user-attachments/assets/a54ad6a7-c6ae-4feb-b2a4-92f0f4fb3455" />

Trying to google what sar2html is, I find that there are multiple articles on a rce exploit abusing it, cool.

While googling trying to find waht version mine was, I found in the github of the toolitself that all versions after 3.2.2 (the version where all previous versions are vulnerable to the rce exploit) used python instead of php, and since sar2html on the webserver was using php, I knew it was vulnerable. 

I searched searchsploit for a script, and there was a python script exploiting the rce vulnerability via the ```?plot=;``` parameter, perfect.

<img width="508" height="168" alt="image" src="https://github.com/user-attachments/assets/c4ae9d58-c351-4794-bfc0-93d6cd65d294" />

Using thsi script, I viewed what was in the current directory, catted out a log txt, and found a valid set of credentials, nice. ヾ(≧▽≦*)o

<img width="922" height="340" alt="image" src="https://github.com/user-attachments/assets/f6dc898d-5659-4598-a971-00d4cd854508" />

Using the valid set of creds ```basterd:superduperp@$$``` I was able to get a shell into the box. 

<img width="776" height="374" alt="image" src="https://github.com/user-attachments/assets/f78030d3-2040-4677-95b5-8d0a640879c7" />

### priv esc

Looking for suid binaries using ```find / -perm -u=s 2>/dev/null | grep /usr/``` I found a binary that usually does not have the suid bit set: the ```find``` command.

<img width="574" height="288" alt="image" src="https://github.com/user-attachments/assets/360179cf-6760-4b54-839d-56fbda54a1a1" />

Looking on gtfobins for the find command, I see an entry to be able to spawn a root shell using the ```find``` command with the SUID bit set by executing ```find . -exec /bin/sh -p \; -quit```.

Using this command, I now have a root shell and root.txt.

<img width="605" height="101" alt="image" src="https://github.com/user-attachments/assets/45861cbb-bf90-4096-89e4-56861d8a543e" />

#### Back to finding user.txt

I accidentally prvilege escalated before finding user.txt, so I ahd to go back and find it.

When I initially enumerated the machine, I had found a bash script that I ahd noted but forgot about once I had found my priv esc vector, but inside there was another user's cerdentials, and it stored a log in the user's home directory.

```bash
REMOTE=1.2.3.4

SOURCE=/home/stoner
TARGET=/usr/local/backup

LOG=/home/stoner/bck.log
 
DATE=`date +%y\.%m\.%d\.`

USER=stoner
#superduperp@$$no1knows

ssh $USER@$REMOTE mkdir $TARGET/$DATE


if [ -d "$SOURCE" ]; then
    for i in `ls $SOURCE | grep 'data'`;do
             echo "Begining copy of" $i  >> $LOG
             scp  $SOURCE/$i $USER@$REMOTE:$TARGET/$DATE
             echo $i "completed" >> $LOG

                if [ -n `ssh $USER@$REMOTE ls $TARGET/$DATE/$i 2>/dev/null` ];then
                    rm $SOURCE/$i
                    echo $i "removed" >> $LOG
                    echo "####################" >> $LOG
                                else
                                        echo "Copy not complete" >> $LOG
                                        exit 0
                fi 
    done
     

else

    echo "Directory is not present" >> $LOG
    exit 0
fi

```

I had searched this directory as root and found nothing and thats when I remembered that hidden files exist.

I searched and found a file called ```.secret``` and voila user.txt was found.

<img width="474" height="166" alt="image" src="https://github.com/user-attachments/assets/c08a5b1a-7312-44af-b7c2-53c4289616c8" />


---

## Solution Steps

1. Enumerate the web server
2. Enumerate the ```/joomla``` subdir
3. Find the ```sar2html``` tool in the ```/_test``` subdir
4. Discover the rce exploit for ```sar2html```
5. exploit it and get creds form log.txt
6. establish an ssh session with creds
7. escalate privileges using ```find``` binary by executing the command ```find . -exec /bin/sh -p \; -quit```



## Thoughts
I am really enjoying these medium rated boxes. For my skill level, tehy are not too hard, yet not too easy. As always, I love exploiting SUID binaries simply becauseof how satisfying ti is to see the root shell prompt appear. But overall an interesting box. Exposed me to the ```sar2html``` rce exploit which was super simple yet interesting to learn about.

---

Thank you for reading!

