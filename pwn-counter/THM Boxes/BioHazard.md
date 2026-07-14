---
title: BioHazard
parent: THM Boxes
grand_parent: Pwn Counter
nav_order: 25
---

# BioHazard

<img width="931" height="270" alt="image" src="https://github.com/user-attachments/assets/4234036e-d284-4d03-99bf-b33372c901be" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> A CTF room based on the old-time survival horror game, Resident Evil. Can you survive until the end?

## Recon

As always, I started with an nmap scan which revealed services on ```ports 21, 22, and 80```.

Visiting the webserver, we are greeted with the start of a story.

<img width="1714" height="799" alt="image" src="https://github.com/user-attachments/assets/f2aba674-85b4-4824-b3e1-2443f0454cb8" />

Clicking the hyperlink, we are redirected to this page with the subdir to visit in the source code: ```/diningRoom```.

Visiting the ```/diningRoom``` subdir we see the continuation of the story as well as this b64 encoded string in the source code: ```SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=```. Decoded, it translates to: ```How about the /teaRoom/```. 

On the page we see an option to take an emblem, going there we get an emblem flag and are instructed to reload the diningRoom page.

<img width="1848" height="773" alt="image" src="https://github.com/user-attachments/assets/a15f1478-f730-4358-8f97-b455971c885b" />

Submitting the emblem flag, we get teh message nothing happen. Nice.

Visiting the ```/teaRoom``` subdir we see this page with a hyperlink to the lockpick flag and a suggested subdir to visit.

Visiting the ```/artRoom``` subdir, we see this page with a hyperlink to a map of the mansion (all the subdirs).

<img width="1565" height="787" alt="image" src="https://github.com/user-attachments/assets/bddc22ca-ea76-4e04-8063-551a76f7eb70" />

<img width="141" height="301" alt="image" src="https://github.com/user-attachments/assets/26096ac0-4ae6-4000-94e7-7115c53f1aa9" />

Going down the list, next up is the ```/barRoom``` subdir.

<img width="1464" height="784" alt="image" src="https://github.com/user-attachments/assets/46b39e68-0c50-4f44-9dba-b8ecc1e387f4" />

Inputting the lockpick flag, we get into the barRoom where we see a hyperlink to a piano music sheet (a base32 encoded string). The string is ``` NV2XG2LDL5ZWQZLFOR5TGNRSMQ3TEZDFMFTDMNLGGVRGIYZWGNSGCZLDMU3GCMLGGY3TMZL5 ``` which base32 decoded is ```music_sheet{362d72deaf65f5bdc63daece6a1f676e}```.

By inputting the flag, we are now in the secret bar room. We first get the gold emblem flag from the hyperlink, and then refreshing asks for the emblem flag from teh dining room emblem.

<img width="1607" height="801" alt="image" src="https://github.com/user-attachments/assets/3db98f6a-6a74-49aa-acc9-d3f181888901" />

Inputting the flag, we get the name ```rebecca```.

Moving down the list we go to the subdir ```/diningRoom2F``` and there we see a rot13 string in the source code: ```Lbh trg gur oyhr trz ol chfuvat gur fgnghf gb gur ybjre sybbe. Gur trz vf ba gur qvavatEbbz svefg sybbe. Ivfvg fnccuver.ugzy``` which decrypted is ```You get the blue gem by pushing the status to the lower floor. The gem is on the diningRoom first floor. Visit sapphire.html```.

Visitin this page, we get the blue jewel flag.

<img width="1567" height="668" alt="image" src="https://github.com/user-attachments/assets/ecee224f-ee25-4eab-987c-40ef41bbb28a" />

So lets go back to the first floor diningRoom.

Inputting the gold emblem flag, we get this string: ```klfvg ks r wimgnd biz mpuiui ulg fiemok tqod. Xii jvmc tbkg ks tempgf tyi_hvgct_jljinf_kvc```. Using dCode, I found the cipher used was called Vigenere and teh decrypted message was ```there is a shield key inside the dining room. The html page is called the_great_shield_key```.

Vistiting the subdir ```/diningRoom/the_great_shield_key.html```, we have gotten the great shield key, but no statue.

Going to the ```/tigerStatusRoom/``` subdir, we are asked to put a jewel in the tiger's eye.

<img width="1448" height="718" alt="image" src="https://github.com/user-attachments/assets/64a9745d-73b6-432d-ac77-14185477b365" />

Putting in the blue jem, we get the following text.

```markdown
crest 1:
S0pXRkVVS0pKQkxIVVdTWUpFM0VTUlk9
Hint 1: Crest 1 has been encoded twice
Hint 2: Crest 1 contanis 14 letters
Note: You need to collect all 4 crests, combine and decode to reavel another path
The combination should be crest 1 + crest 2 + crest 3 + crest 4. Also, the combination is a type of encoded base and you need to decode it
```

So this means that we need to find 3 more crests.

Going to the ```/galleryRoom``` subdir we find this page.

<img width="1437" height="749" alt="image" src="https://github.com/user-attachments/assets/6e0a4f0d-d731-40dd-b64a-ebb352bc0387" />

The hyperlink takes us to teh second crest.

```markdown
crest 2:
GVFWK5KHK5WTGTCILE4DKY3DNN4GQQRTM5AVCTKE
Hint 1: Crest 2 has been encoded twice
Hint 2: Crest 2 contanis 18 letters
Note: You need to collect all 4 crests, combine and decode to reavel another path
The combination should be crest 1 + crest 2 + crest 3 + crest 4. Also, the combination is a type of encoded base and you need to decode it
```

I tried the studyRoom subdir but I need the helmet crest so I went to the armorRoom and input the shield flag.

This new room takes us to the third crest via hyperlink.

```markdown
crest 3:
MDAxMTAxMTAgMDAxMTAwMTEgMDAxMDAwMDAgMDAxMTAwMTEgMDAxMTAwMTEgMDAxMDAwMDAgMDAxMTAxMDAgMDExMDAxMDAgMDAxMDAwMDAgMDAxMTAwMTEgMDAxMTAxMTAgMDAxMDAwMDAgMDAxMTAxMDAgMDAxMTEwMDEgMDAxMDAwMDAgMDAxMTAxMDAgMDAxMTEwMDAgMDAxMDAwMDAgMDAxMTAxMTAgMDExMDAwMTEgMDAxMDAwMDAgMDAxMTAxMTEgMDAxMTAxMTAgMDAxMDAwMDAgMDAxMTAxMTAgMDAxMTAxMDAgMDAxMDAwMDAgMDAxMTAxMDEgMDAxMTAxMTAgMDAxMDAwMDAgMDAxMTAwMTEgMDAxMTEwMDEgMDAxMDAwMDAgMDAxMTAxMTAgMDExMDAwMDEgMDAxMDAwMDAgMDAxMTAxMDEgMDAxMTEwMDEgMDAxMDAwMDAgMDAxMTAxMDEgMDAxMTAxMTEgMDAxMDAwMDAgMDAxMTAwMTEgMDAxMTAxMDEgMDAxMDAwMDAgMDAxMTAwMTEgMDAxMTAwMDAgMDAxMDAwMDAgMDAxMTAxMDEgMDAxMTEwMDAgMDAxMDAwMDAgMDAxMTAwMTEgMDAxMTAwMTAgMDAxMDAwMDAgMDAxMTAxMTAgMDAxMTEwMDA=
Hint 1: Crest 3 has been encoded three times
Hint 2: Crest 3 contanis 19 letters
Note: You need to collect all 4 crests, combine and decode to reavel another path
The combination should be crest 1 + crest 2 + crest 3 + crest 4. Also, the combination is a type of encoded base and you need to decode it
```

Finally, the attic leads us to the final crest piece.

```markdown
crest 4:
gSUERauVpvKzRpyPpuYz66JDmRTbJubaoArM6CAQsnVwte6zF9J4GGYyun3k5qM9ma4s
Hint 1: Crest 2 has been encoded twice
Hint 2: Crest 2 contanis 17 characters
Note: You need to collect all 4 crests, combine and decode to reavel another path
The combination should be crest 1 + crest 2 + crest 3 + crest 4. Also, the combination is a type of encoded base and you need to decode it
```
All four crests merged together lead to the encoded base64 string of ```RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==``` which decoded is ftp creds, nice. ```FTP user: hunter, FTP pass: you_cant_hide_forever```.

From the ftp server we get the encrypted helmet key, 3 numbered key pictures, and a file called ```important.txt``` which says

```markdown
Jill,

I think the helmet key is inside the text file, but I have no clue on decrypting stuff. Also, I come across a /hidden_closet/ door but it was locked.

From,
Barry
```

Using stegseek on the first key image we get the first part of the key, then using strings on th second image we get its part, and then binwalk gives us teh final part for the last image. The final key is: ```plant42_can_be_destroy_with_vjolt```.

Using that key we can decrypt the gpg file and get the helmet key.

Going to the ```/hidden_closet``` subdir we input the helmet key and we get the ssh password of ```T_virus_rules``` and the follwoing encrypted text: ```wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk```. This is another vigenere cipher which decoded is ```	weasker login password, stars_members_are_my_guinea_pig```.

Going to the studyRoom subdir then gives us an ssh user after some gunzip and tar extracting: ```SSH user: umbrella_guest```.

With this we now have the ssh creds of ```umbrella_guest:T_virus_rules```.

Finding out weasker was the traitor through the story in the terminal, I log in as weasker with teh traitor's password, he can run anythin as sudo, so I read root.txt and the box is finished.

### priv esc



---

## Solution Steps

1. read the writeup



## Thoughts
This was a fun cryptography box. I dotn feel like I learned anything or sharpened my skills, but it was fun nontheless.

---

Thank you for reading!

