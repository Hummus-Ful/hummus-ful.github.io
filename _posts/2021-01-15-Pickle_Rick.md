---
title: Pickle Rick  
categories: [Try-Hack-Me] # Up to two elements only
tags: [tryhackme, easy, pickle rick, ctf]     # TAG names should always be lowercase, infinate number of elements
image: /assets/img/posts/Pickle_rick/Pickle_rick.webp    # If you want to add an image to the top of the post contents
# toc: false    # table of content - overwrite global configuration from _config.yml
# comments: flase       # overwrite global configuration from _config.yml
# pin: true     # pin one or more posts to the top of the home page
excerpt_separator: <!--exc-->
permalink: /:categories/:year/:month/:day/:title:output_ext
published: true
---

This is a writeup for the Pickle Rick theme challenge on Try-Hack-Me which requires you to exploit a Webserver to find 3 ingredients that will help Rick make his potion to transform himself back into a human from a pickle. Rated as Easy/Beginner level machine.
<!--exc-->


# Introduction
In this post, we'll try to root [Pickle Rick](https://tryhackme.com/room/picklerick). It was created by [tryhackme](https://tryhackme.com/p/tryhackme). It is rated as **Easy/Beginner** level machine.


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
$ mkdir ~/tryhackme/pickle_rick
$ cd ~/tryhackme/pickle_rick/
```


# Scanning
## Website
Using Firefox we can check the website on port 80. It is a Rick and Morty theme where the index page has one photo and text stating we need to look for three ingredients

![try-hack-me pickle rick homepage](/assets/img/posts/Pickle_rick/try-hack-me-pickle-rick-homepage.webp) _Pickle Rick homepage_

Nothing intresting to see here. Let's view the page source (right click on the page -> View Page Source). Here we find a username

![try-hack-me-pickle-rick-comment-view-page-source](/assets/img/posts/Pickle_rick/try-hack-me-pickle-rick-comment-view-page-source.webp) _Pickle Rick comment in page source_

There are no unusual HTTP headers.

## nikto
start nikto scan:

```console
$ nikto -h http://10.10.128.137/
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.128.137
+ Target Hostname:    10.10.128.137
+ Target Port:        80
+ Start Time:         2021-01-07 18:19:04 (GMT-5)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /, inode: 426, size: 5818ccf125686, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ Cookie PHPSESSID created without the httponly flag
+ OSVDB-3233: /icons/README: Apache default file found.
+ /login.php: Admin login page/section found.
+ "robots.txt" contains 1 entry which should be manually viewed.
```

Two interesting paths:
* **robots.txt** - in this case the file contains only one string, nothing else. Might be useful in the future.
* **/login.php** - the Portal login page. 


# Gaining Access
We can try to use the username we found in the page source and combine it with the string from the robots file.

![try-hack-me-pickle-rick-login-page](/assets/img/posts/Pickle_rick/try-hack-me-pickle-rick-login-page.webp) _Pickle Rick login page_

Success! we have access to the portal.

## Command Panel
The portal main page is a Command Panel which allow us to run Unix commands on the backend (server).
Let's try simple command - `whoami` which returns `www-data`, the user running the web service.
If we issue a `ls` command we get a list of files where the first one looks like the one we're after

```console
[REDUCTED] 
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

As you recall, `robots.txt` is accessible directly using a browser. Therefore we can assume the directory itself and all of the files should be accessible. We'll try `http://10.10.70.251/REDUCTED`

Success! The file content is the first ingredient we need.

- [x] First Flag
- [ ] Second Flag
- [ ] Third Flag

## find
If you try to access the rest of the files in that directory you'll find a file stating **"Look around the file system for the other ingredient."**. To do so we can use the `find` command to look for other files related to the ingredients. 
**Keep in mind** - The search is limited by user permissions, yet we should give it a try. The command is `find / -iname "*ingred*" -type f &2>/dev/null` 
* `/` is the root path to start the scan from
* `-type f` search for file (d for directory)
* `-iname "*ingred*"` file name to loog for. We also use wildcards (`*`) as the filename might start/end with additional string.
* `&2>/dev/null` this will produce a cleaner output as it will discard errors, such as permission 

![try-hack-me-pickle-rick-find-search-results](/assets/img/posts/Pickle_rick/try-hack-me-pickle-rick-find-search-results.webp) _Pickle Rick find command results_

We know the first file, but the second one is new. 

## cat / head / tail / less
The `cat`, `head` and `tail` commands are disabled. `less`, on the other hand is enabled and we're able to print the file content using `less /home/rick/"REDUCTED"`

Note:
: The filname includes a space, therefore we must use quotes on the filename.

Success! we have the second ingredient.
- [x] First Flag
- [x] Second Flag
- [ ] Third Flag


# Privilege Escalation

Note:
: Links to further reading about `sudo -l` can be found at the Summary

We can use the `sudo -l` command to list which commands we can run as `sudo`:

```console
Matching Defaults entries for www-data on ip-10-10-230-106.eu-west-1.compute.internal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-10-230-106.eu-west-1.compute.internal:
    (ALL) NOPASSWD: ALL
```

Amazing! we can run all (any) commands as sudo. Nice. let's investigate the root folder using `sudo ls /root`

```console
[REDUCTED]
snap
```

OK, Let's pring the first file using `less`: 
`sudo less /root/RDUCTED`

Now we have the third ingredient.

- [x] First Flag
- [x] Second Flag
- [x] Third Flag


# Potential Rabbit Holes
## Steganography
Nikto found the /assets directory which contains multiple images. My first instinct (and maybe yours as well) is to look for hiden data in these files. Not in this case. 


# Summary
* Look into website source. F12 and Developer Tools are your friends.
* If one Unix command is being blocked or disabled, try to find another command to use e.g. `cat` and `less`.

## Reading Materials
* *sudo -l* - I see this technice used in many CTFs, It's simple to run and easy to understnad. Make sure you feel comfortable with it. [READ](https://www.explainshell.com/explain?cmd=sudo+-l), [READ2](https://medium.com/better-programming/becoming-root-through-misconfigured-sudo-7b68e731d1f5)
