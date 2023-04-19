+++ 
title = "Knife HTB Write up"
description = "Write up for the HTB machine 'Knife'"
author = "greatmoves"
tags = ["php", "sudo -l", "gtfobins", "hackthebox"]
date = 2023-03-01
+++
- [1. Recon](#1-recon)
  - [1.1. nmap](#11-nmap)
  - [1.2. nikto](#12-nikto)
- [2. Privilege escalation](#2-privilege-escalation)

## 1. Recon
----
### 1.1. nmap
A simple nmap scan: `nmap -sC -sV {target-ip}` reveals port 22-ssh and port 80-http.

### 1.2. nikto
A nikto scan using the command `nikto -host {target-ip}` reveals that the web application is using `PHP/8.1.0-dev`.

A quick google search of this PHP version shows us a Backdoor Remote Code Execution script found at:
https://github.com/flast101/php-8.1.0-dev-backdoor-rce

Run the script along with a netcat listener and catch your reverse shell!

Navigate to /home/james for user.txt.

## 2. Privilege escalation
----
As our user let's see if there's anything they are able to run as root using the command `sudo -l`

which reveals:
```
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

Looking up "knife" on https://gtfobins.github.io/ we find the command `sudo knife exec -E 'exec "/bin/sh"'`, so let's run it.

root :)

Navigate to /root for root.txt