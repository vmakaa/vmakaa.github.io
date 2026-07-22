---
title: Overpass 3
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 29
---

# Overpass 3

<img width="954" height="280" alt="image" src="https://github.com/user-attachments/assets/cf4ef164-ea9d-401b-9685-1bffdede88b4" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> You know them, you love them, your favourite group of broke computer science students have another business venture! Show them that they probably should hire someone for security...

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 21, 22, and 80```. 

Visiting the webpage, we see a webpage for a hosting solution.

I try checking the source code for any hidden info, but nothing.

I ran cmseek and nuclei to no avail, but my gobuster scan revealed one subdirectory: ```/backups```.

Going into the the ```/backups``` directory, I see it is an exposed file listing with a ```backups.zip```.

Downloading it to my machine, I unzip and get a gpg encrypted spreadsheet and a private key.

Right off the bat  I tried to see of this private key would be able to decrypt the spreadsheet.

I used ```gpg``` to import the key into my keyring, directed the output of the decrypted file content to another file, and opened it with libreoffice since it was a spreadsheet.

Here I see a list of usernames and passwords along with credit card details.

<img width="674" height="78" alt="image" src="https://github.com/user-attachments/assets/fb5f25f0-45cf-42a0-93cb-02e82e6b71ec" />

### Checking for password reuse

With 4 sets of credentials, I start checking for password reuse.

I start with ```paradox:ShibesAreGreat123```.

I tried them with ssh but didnt get a session established, but got one with ftp, nice.

### Inside FTP

I looked around for any useful info and found a .jpg file named ```hallway.jpg```. I tried checking for steganography using ```steghide and stegseek```, but I think that was a dead end.

I then realized that the ftp root directory's contetns were exactly what was on teh webpage, we essentially had a shell with only file download and upload capabilities.

### Initial Foothold

Knowing this, I created a new directiory ```a``` and uploaded my php reverse shell. 

Navigating to the shell in the webpage, we get a callback on my listener, nice.

<img width="900" height="179" alt="image" src="https://github.com/user-attachments/assets/ed45ce82-470c-4c45-9588-daf18dbbbf0a" />

### SSH Session

After enumerating around the box a bit as the ```apache``` user, I cannot find any low hanging fruit like SUID binaries or exploitable crontabs. 

When trying to go to the home directory to look at the users on the box, I see the paradox user and check for password reuse by doing ```su paradox``` and Success! I now have a privileged user shell.

I checked my sudo permissions but unfortunatly was not able to run sudo at all.

Enumerating in my home directory, I find that I have access to my .ssh folder. 

I created a keypair on my attack box, added my public key to the authorized_keys file, tried to ssh in with the key, and got a valid ssh session, nice.

### Further Enum

Now that I have access to a stable ssh session, I decided to do some more enumeration with linpeas.

After scrolling through the output for a while, I notice this line.

<img width="466" height="117" alt="image" src="https://github.com/user-attachments/assets/40e309e0-7706-4acb-b52a-221c9968172f" />

This does not look like a defualt system service.

I sart googling how to mount the drive to my attackbox and found that my box wasnt able to mount it due to a route not being found.

I accidentally did the command ```showmount -e box_ip``` on the box and saw the ```/home/james``` share, but realized I had meant to do that on my attack machine adn did teh same command, but this time it gave me the error ```RPC: Unable to recieve```.

Huh. It looks like this share is only accessible locally. Lets try and forward the port the share is on to our local machine with ssh.

### Port Forwarding the Share

I do the command ```ssh -i paradox -L 192.168.130.135:7777:10.66.183.175:2049 paradox@10.66.183.175``` and get a valid ssh session.

On my attack machine I do ```ss -tulpn``` to confirm ```port 7777``` is active and it is, success!

<img width="1791" height="128" alt="image" src="https://github.com/user-attachments/assets/bfc858a7-480c-4128-b674-d19351bfdd1d" />

After some googling on how to mount a local nfs share, I do the command ```sudo mount -t nfs4 -o port=7777 192.168.130.135:/ /home/user/nfs``` and succesfully mount the share and get the user flag.

<img width="639" height="266" alt="image" src="https://github.com/user-attachments/assets/1e19b056-52a4-4812-abc3-b8ab28911e1c" />

### SSH Session as James

Looking in the hidden directory on the mounted drive, I find the user james' public and private key and am able to ssh with the private key into the box as the james user.

### priv esc

Since ```no_root_squash``` is active on teh drive, any file that I write as root on my attack box to the shared drive, will also appear in teh original drive on the box.

I escalated to root privileges on my attack box, copied my bash binary and set the suid bit, and tehn copied it to teh share drive to try and execute it as james, but I realized from teh errors that i recieved that my bash binary was too new for the remote box.

So, I copied the bash binary from the remote box into the shared drive, then on my box I changed the bash binary's owner and group as root:root and then I set the SUID bit.

On the remote box, I eecuted the bash binary on the shared drive using ```./bash -p``` and i got a root shell and the root flag, nice.

<img width="613" height="277" alt="image" src="https://github.com/user-attachments/assets/7a6d4b67-e022-4995-9fc5-09cfc5482c1b" />

---

## Solution Steps

1. Find the ```/backups``` directory and download ```backups.zip```
2. Decypt the gpg file with the key from the zip and open the file with libreoffice to find some credentials
3. Use paradox's credentials to log into FTP
4. Realize that the FTP share is the web server's directory, so you can create a directoty, upload a reverse shell, and naviagte to it in your browser to activate a reverse shell
5. Check for password reuse and escalate to paradox's account
6. Create an ssh key pair and put your public key into paradox's ```authorized_keys``` file and get an ssh session
7. Using linpeas, find that james' nfs in directory ```/home/james *``` has ```no_root_squash``` enabled
8. Find out that the share is only accessible locally
9. Use ssh to port forward the port the share is on to your attack machine
10. mount the shared drive
11. copy the bash binary from the box to the shared drive directory
12. on your attack machine as root, set the bash binary's file owner and group to root and set the SUID bit
13. Then finally on the box as the james user execute the bash binary with ```./bash -p``` and gte a root shell

## Thoughts
I really enjoyed the priv esc technique this box featured. Prior, I wasa not familiar with nfs drives and exploiting them. But, now I feel if I see it again I will be prepared, and I always appreciate when a box teaches me a new, useful skill in preparation for the OSCP. I though the initial foothold was pretty creative too, it took me a while to realize that the ftp share directory was teh web directory and it seemed so obvious once I realized. Overall a great and fun box. Definitly reccomend.

---

Thank you for reading!

