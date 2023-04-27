+++ 
title = "Blocky HTB Write up"
description = "Write up for the HTB machine 'Blocky'"
author = "greatmoves"
tags = ["wordpress", "sudo -l", "dirbuster", "wpscan", "minecraft" , "hackthebox", "linux"]
date = 2023-03-03
+++
- [1. Recon](#1-recon)
  - [1.1. nmap](#11-nmap)
  - [1.2. nikto](#12-nikto)
  - [1.3. wpscan](#13-wpscan)
  - [1.4. dirbuster](#14-dirbuster)
  - [1.5. jd-gui](#15-jd-gui)
- [2. Privilege escalation](#2-privilege-escalation)

## 1. Recon
----
### 1.1. nmap
cmd: `nmap -sC -sV {target-ip}`

Our nmap scan reveals port port 21-ftp, 22-ssh and port 80-http.

### 1.2. nikto
cmd: `nikto -host {target-ip}`

Our nikto scan reveals the site is vulnerable to XSS
```
/modules.php?letter=%22%3E%3Cimg%20src=javascript:alert(document.cookie);%3E&op=modload&name=Members_List&file=index: Post Nuke 0.7.2.3-Phoenix is vulnerable to Cross Site Scripting (XSS).
```

It also reveals the url for the web application, `blocky.htb`, so let's add that to /etc/hosts

### 1.3. wpscan
cmd: `wpscan --url http://blocky.htb -e u `

Scrolling to the bottom of the web app we can see that it is powered by WordPress, so let's run wpscan

from wps scan we can see that there is a user `notch`

### 1.4. dirbuster
cmd: `dirb http://blocky.htb`

since we know it is a WordPress site, let's also run dirbuster to see what else we can find

after the scan has run let's enumerate through the directories it found.

at `/plugins` there are two files that we can download

### 1.5. jd-gui
cmd: `jd-gui BlockyCore.jar`

using `jd-gui` we can decompile both of the `.jar` files

in `BlockyCore.jar` we can see `BlockyCore.class` where we can find the password `8YsqfCTnvxAUeduzjNSXe22`

from there we are able to ssh in to the machine with the credentials
```
notch:8YsqfCTnvxAUeduzjNSXe22
```

## 2. Privilege escalation
----
`sudo -l` reveals:

```
User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
```

which is not that exciting because we just enter the command `sudo su` and we are root!

navigate to `/home/notch` for `user.txt` and `/root` for `root.txt`.