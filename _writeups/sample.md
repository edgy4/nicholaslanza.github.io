---
layout: single
title: 'Hack Test'
collection: writeups
author_profile: true
date: 2012-08-14
permalink: /writeups/exploiting-ad-misconfigurations/
tags:
  - cool-posts
  - category1
  - category2
---

This is a sample blog post. Lorem ipsum I can't remember the rest of lorem ipsum and don't have an internet connection right now. Testing testing testing this blog post. Blog posts are cool.

Headings are cool
======

You can have many headings
======

Aren't headings cool?
------

# Exploiting Active Directory Misconfigurations

## Overview

In this writeup, I document how I identified and exploited misconfigurations in an Active Directory lab environment. This includes enumeration techniques, privilege escalation paths, and mitigation recommendations.

## Lab Setup

- Windows Server 2019 domain controller  
- Three Windows client machines  
- Lab user accounts: `alice`, `bob`, `charlie`  

## Steps Taken

1. **Enumeration**  
   - Used `enum4linux` and `BloodHound` to identify AD users and permissions.  
   - Discovered `alice` had delegated admin rights on a sensitive group.

2. **Privilege Escalation**  
   - Leveraged misconfigured ACLs to escalate from `alice` to `Domain Admin`.  
   - Exported hashes using `mimikatz` for demonstration purposes.
