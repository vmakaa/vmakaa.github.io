---
title: Nexus
parent: HTB Boxes
grand_parent: Pwn Counter
nav_order: 28
---

# Orion

<img width="845" height="525" alt="image" src="https://github.com/user-attachments/assets/db087a49-1daf-4fa9-88f6-9d71f397b425" />


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Box Description

> Easy HTB Machine

## Recon

As always, I started off with an nmap scan which revealed services on ```ports 22 and 80```. 

I visited the webpage and it was a website for an energy company.

<img width="1867" height="752" alt="image" src="https://github.com/user-attachments/assets/cc911bb8-8c8e-442c-9958-d40576deccad" />

Towards the bottom we see a valid username we can save just in case.

<img width="757" height="207" alt="image" src="https://github.com/user-attachments/assets/db961e4f-bb6c-42e6-a4bb-e2b69d14c35e" />

---

I tried enumerating using cmseek, gobuster, and nuclei, but found nothing of interest.

I decided to finally learn how to fuzz web urls.

Fuzzing the ```http://nexus.htb```, we get two subdomains ```git.nexus.htb``` and ```billing.nexus.htb``` which is a gittea repository and a kramin CRM login page respectively.

---

In the Gitea repository we see a ```docker-compose``` file as well as a ```.env``` file where the author forgot to remove a password in his first commit.

<img width="379" height="191" alt="image" src="https://github.com/user-attachments/assets/5f5068ff-0f8e-486f-b3b3-a2a14b8336c8" />

---

Using the email from earlier and the password we just found, I tried logging into the login panel at ```billing.nexus.htb``` and we are in!

Upon visiting the kraymin crm we see we are on version ```2.2.0```.

---

## Initial Foothold

Doing a quick google search, we see that kraymin crm v 2.2.0 is vulnerable to authenticated RCE via unrestricted php file upload with CVE-2026-38526 assigned to it.

Using this [exploit](https://www.exploit-db.com/exploits/52629) I found on exploit-db, I successfully upload a php-reverse-shell to the endpoint vulnerable and get a nice shell.

<img width="1073" height="526" alt="image" src="https://github.com/user-attachments/assets/9059ed25-0b0d-4b73-9ef3-a36f2d78291e" />

---

Enumerating around, I find a ```.env``` file in the ```/krayin``` directory on the box and find credentials for the MySQL server.

<img width="229" height="129" alt="image" src="https://github.com/user-attachments/assets/72ad6757-0bce-43c5-aec8-12e4918e1235" />

---

## Established SSH session as jones user

After bring stuck for a while I found out the intended path was to try the password against the ```jones``` user on the box.

<img width="501" height="56" alt="image" src="https://github.com/user-attachments/assets/d54760b7-7485-4b85-a20f-3ad3061ffd68" />

---

## Priv Esc

As a discalimer, I did use AI and a [writeup](https://exploitnotes.hashnode.dev/hackthebox-nexus-writeup#step-4-manual-exploitation) to help me manually exploit this. Very niche and time consuming priv esc, but cool.

---

When running my linpeas scan, I found that there was an unusual timer with root privileges that was running a python script every minute.

When I looked inside the python script, I tried checking for low hanging fruit like module hijacking or the script being world writable, but unfortunatly, I came accross nothing.

---

Trying to better understand the script, I fed it to AI to get some more pratice in code auditing.

I realized that the script was getting an API token, creating a template repo, listing the commits in the template repo of type blob which means its a singular file, splitting on the metadata and filename, then the script takes whatever was commited to the template repo and syncs with the box by taking the altest commits and writing it to the equivalent location on the box.

This is a normal syncing script besides the fact when the script creates the filepathto be written to, no checks are performed on whether ot not the filepath is legit or even if it still has the same base path.

So, for example, if the file name was ```../../../../../../../../root``` you are now writing in the root directory instead of the intended directory.

The script is shown below:

```python
import os
import sys
import json
import subprocess
import time
import urllib.request

GITEA_URL = "http://localhost:3000"
REPO_ROOT = "/var/lib/gitea/data/gitea-repositories"
STAGING_DIR = "/home/git/template-staging"
LOG_FILE = "/var/log/template-sync.log"

def log(msg):
    ts = time.strftime("%Y-%m-%d %H:%M:%S")
    line = "[%s] %s" % (ts, msg)
    print(line, flush=True)
    try:
        os.makedirs(os.path.dirname(LOG_FILE), exist_ok=True)
        with open(LOG_FILE, 'a') as f:
            f.write(line + '\n')
    except:
        pass

def load_config():
    config = {}
    for path in ['/etc/gitea/template-sync.conf', '/opt/forge/app/.env']:
        try:
            with open(path) as f:
                for line in f:
                    line = line.strip()
                    if line and not line.startswith('#') and '=' in line:
                        k, v = line.split('=', 1)
                        config[k.strip()] = v.strip()
        except:
            pass
    return config

def get_token():
    cfg = load_config()
    return cfg.get('GITEA_API_TOKEN')

def get_template_repos(token):
    url = "%s/api/v1/repos/search?limit=50" % GITEA_URL
    req = urllib.request.Request(url, headers={
        'Authorization': 'token %s' % token
    })
    try:
        with urllib.request.urlopen(req) as resp:
            data = json.loads(resp.read())
            repos = data.get('data', data) if isinstance(data, dict) else data
            return [r for r in repos if r.get('template', False)]
    except Exception as e:
        log("API error: %s" % e)
        return []

def sync_template(repo_info):
    owner = repo_info['owner']['login']
    name = repo_info['name'].lower()
    bare_path = os.path.join(REPO_ROOT, owner, "%s.git" % name)
    stage_path = os.path.join(STAGING_DIR, owner, name)

    if not os.path.isdir(bare_path):
        log("  repo not found: %s" % bare_path)
        return

    # Read tree entries from the bare repository
    try:
        GIT = ['git', '-c', 'safe.directory=*']
        result = subprocess.run(
            GIT + ['ls-tree', '-r', 'HEAD'],
            cwd=bare_path,
            capture_output=True, text=True, timeout=10
        )
        if result.returncode != 0:
            log("  ls-tree failed: %s" % result.stderr.strip())
            return
    except Exception as e:
        log("  ls-tree error: %s" % e)
        return

    entries = []
    for line in result.stdout.strip().split('\n'):
        if not line:
            continue
        parts = line.split('\t', 1)
        if len(parts) != 2:
            continue
        meta, filepath = parts
        mode, objtype, objhash = meta.split()
        if objtype == 'blob':
            entries.append((mode, objhash, filepath))

    if not entries:
        log("  no files in template")
        return

    # Extract files to staging directory
    for mode, objhash, filepath in entries:
        target = os.path.join(stage_path, filepath)
        target_dir = os.path.dirname(target)

        try:
            os.makedirs(target_dir, exist_ok=True)
            GIT = ['git', '-c', 'safe.directory=*']
            cat_result = subprocess.run(
                GIT + ['cat-file', 'blob', objhash],
                cwd=bare_path,
                capture_output=True, timeout=10
            )
            if cat_result.returncode != 0:
                continue

            with open(target, 'wb') as f:
                f.write(cat_result.stdout)

            if mode == '100755':
                os.chmod(target, 0o755)
            else:
                os.chmod(target, 0o644)

            log("  synced: %s" % filepath)
        except Exception as e:
            log("  error syncing %s: %s" % (filepath, e))

def main():
    log("Template sync starting")

    token = get_token()
    if not token:
        log("No API token found")
        sys.exit(1)

    templates = get_template_repos(token)
    log("Found %d template repo(s)" % len(templates))

    for repo in templates:
        name = repo['full_name']
        log("Syncing template: %s" % name)
        sync_template(repo)

    log("Template sync complete")

if __name__ == '__main__':
    main()
```
---

### Step 1: Create the ssk key pair to use

Now the plan of action is to generate a ssh key pair for the jones user and write that into the ```/root/.ssh/authorized_keys``` directory.

In order to create a ssh key pair I do the command ```ssh-keygen -t ed25519 -f /tmp/jsk -N ""```. 

I now have the contents of the public key which is ```ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKTHBCMDwYKKtyu0Lwlzf/gE/A2hsxb5pI76fYBELqlO jones@nexus```

---

### Step 2: Get the valid API toekn for gitea

To get a valid API token, I go to ```git.nexus.htb```, login with jones' creds of ```jones:y27xb3ha!!74GbR```, go to generate a token in settings with every single permission enabled for the token, and I get this token: ```2e63b969a31b88b20330b75d2f81e23ae1f96854```.

---

### Step 3: Create a fake repo and mark it as a template

Using the UI, I created a template repo.

### Step 4: git push directly to object files to get directory traversal write to root ssh authorized keys

In order to do this, the official writeup provided a script to use that would make the file name and do the git magic to download the file and directory traverse from the timer to get your key added to authorized keys. 

### Step 5: To root

Use your private key to ssh as root into the box and get your private key.

<img width="957" height="745" alt="image" src="https://github.com/user-attachments/assets/7255c45d-6e71-4df8-8db2-b4030e0e6732" />

---


## Solution Steps

1. Use ffuf to find the ```billing.nexus.htb and git.nexus.htb``` subdomains
2. Explooit kraymin v 2.2.0 to perform authenticated rce using uploads vulnerability to upload reverse shell
3. Enumerate and locate the ```.env``` file with the user ```jones``` password
4. ssh as jones into the box
5. find adn read the python script for the gitea sync script
6. find the file traversal vulnerability in the source code
7. use the script to upload an object file containing your public ssh key named ```../../../../../root/.ssh/authorized_keys``` to your template gitea repo
8. the gitea timer sync script will download that and due to the vulnerability in the python source code, will actually write to ```/root/.ssh/authorized_keys``` instead of the intended hardcoded directory
9. Wait for gitea sync script to run adn tehn ssh with private key into root on the box

## Thoughts
This was a super super hard box. It made me realize that I definitly need to get comfortable scripting with python. It also made me realize that maybe I should try some retired boxes instead of live boxes to accurately see what the community thinks the difficulty is. The foothold was fun toe xecute, but the priv esc was really challenging, though I feel I got a little better at source code auditing. I think this box was a bit above my skill level, so I was not really able to learn and do it myself when it came to the priv esc.

--

Thank you for reading!

