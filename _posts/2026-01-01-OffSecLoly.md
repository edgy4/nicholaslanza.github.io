---
layout: single
title: 'Proving Grounds Loly'
author_profile: true
date: 2012-08-14
permalink: /writeups/loly/
tags: [proving-grounds, linux, wordpress]
---



## Enumeration
The attacker starts by checking available services.


```bash 
sudo nmap -sC -sV  -p- $IP --open            
[sudo] password for nick: 
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

![nmap scan](/images/loly/nmap.png "nmap scan")


## Exploitation 
Test

## Privlege Escalation
