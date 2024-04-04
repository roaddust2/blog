---
title: '[Infrastructure] First VPS setup after purchase'
description: I'll tell you how I set up a fresh server under Ubuntu 22.04
slug: first-vps-setup-after-purchase

categories: ["Infrastructure"]
tags: ["VPS", "Server", "Ubuntu 22.04"]

date: 2024-03-31T12:56:19+07:00
draft: false

cover:
  image: "cover.jpg"
  alt: "Meme about VPS: Low IQ - I put everything on VPS; Average IQ-Serverless/Lambdas/etc.; High IQ - I put everything on VPS"
  relative: true
---

## Introduction
> :v: The author's opinion may differ from yours and that's okay.

I decided to devote the first post to setting up a VPS (Virtual private server). I think there’s no need to explain for a long time what it is and what it’s needed for, ~~of course, to set up your Minecraft server and install a VPN~~. There can be many tasks, hosting static files, some applications, launching containers, or just practicing administration, but the setup, plus or minus, is the same.

But what can be done to ensure that the server is minimally resistant to unauthorized access and does not have a thousand miners installed on it? Now I'll show you how I do it. Using Ubuntu 22.04 OS, let's get started.

## First thing
To begin with, I connect to the server using the log/passes provided by the hosting and download the latest updates:

```bash
ssh root@your_server_ip
```

```bash
apt update
apt upgrade
```

You may need to reboot, you can do it with one command, and then log in again:

```bash
reboot
```

Working as root is not very safe, so you need to create a user. Create a user named "admin". We give it a password, the rest is optional, we delegate administrator rights (sudo).

```bash 
adduser admin
usermod -aG sudo admin
```

## Security

To protect our server a little bit, I do three things:
- Setting up SSH keys
- I prohibit password authentication
- Setting up a firewall, in our case ufw

### SSH Keys
  
Let's start with SSH, it is necessary so that in the future you can log into the server from a machine with a trusted certificate; the advantage is that there is no need to enter a password each time.

Let's go to the local machine and generate a key pair.
The -t flag specifies the key type, I choose "ed25519", and -f allows us to select the directory where to place the pair and the name of the key. I leave the "Enter passphrase" field empty. 

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vps
```

We send the created public key to the server for the root user using the "ssh-copy-id" utility. The -i flag is required to select a specific key. What is the -o flag for? It is not obvious, but when you try to add a key to the server, ssh-copy-id tries to log in to the server using the same key, although it is not installed yet, this leads to the error “Too many authentication failures”. Therefore, we use the flag for password authorization.

```bash
ssh-copy-id -i ~/.ssh/vps.pub -o PubKeyAuthentication=no root@your_server_ip
```

Let's check if everything works (if it doesn't ask for a password, everything is ok):

```bash
ssh root@your_server_ip
```

Yes, now we can log into the server without a password. We add a key for our user “admin”, I do this via rsync (don’t forget to set the owner and group admin:admin). Let's go out.

```bash
apt install rsync
rsync --archive --chown=admin:admin ~/.ssh /home/admin
exit
```

### Prohibition of password authorization

> :warning: Before going further, make sure that you can access the server without a password using the SSH key, otherwise you will lose access to your VPS! Well, I warned you, let's move on.

To ensure that the password cannot be guessed and authorization is only available using SSH keys, you must disable the ability to log in using a password. Makes sense? Absolutely.

Login through our user. At this point, the password is no longer requested, because we added the keys earlier.

```bash
ssh admin@your_server_ip
```

Open sshd_config:

```
sudo nano /etc/ssh/sshd_config
```

Change the following lines from yes to no:

```
PasswordAuthentication no
UsePAM no
PermitRootLogin no
```

Restart the ssh service for the changes to take effect:

```bash
sudo systemctl reload ssh
```

### Firewall ufw

We install all the necessary programs, I will make do with a proxy server and certbot to install Let's Encrypt certificates. We also install the firewall itself.

```bash
sudo apt install nginx certbot python3-certbot-nginx
sudo apt install ufw
```

Now let’s add rules to our ufw firewall (a wrapper over ip tables). We want to deny entry to everything except access via SSH, and we also distribute Nginx access.

Let's check if ufw is working:

```bash
sudo ufw status
```

It should be like this:

```bash
Status: inactive
```

We distribute access and launch our firewall:

```bash
sudo ufw allow OpenSSH
sudo ufw allow "Nginx Full"
sudo ufw enable
```

```bash
Command may disrupt existing ssh connections. Proceed with operation (y|n)? # y
Firewall is active and enabled on system startup
```

Checking access:

```bash
sudo ufw status
```

```bash
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx Full                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx Full (v6)            ALLOW       Anywhere (v6)             
```

Now we only have access to an SSH connection and access via the Internet via port 80 `http://your_server_ip/`, there will be an nginx welcome page.

Done!
