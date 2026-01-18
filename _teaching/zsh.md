---
layout: single
title: "Saving Time on Engagements With ZSH Automation"
collection: teaching
categories: teaching
permalink: /teaching/zsh-workflow/
tags: [zsh, tutorials]
---

## Overview

During engagements, a significant amount of time is lost repeatedly typing the same commands, setting target IPs, and running scans.

This tutorial is made to show how small zsh customizations can save you a ton of time on your next engagement.



## What is the .zshrc file and Where is it Located 
The .zshrc file is the configuration file for the Z shell, the default shell in Kali Linux and Parrot OS. It contains settings, environment variables, and customizations that load every time you start a new terminal session. By default, the file is located in your home directory at `$HOME/.zshrc` ($HOME refers to your home directory).
## Defining the Target Once
Instead of repeatedly typing a target machine's IP, define it once at the start of your engagement by adding `export IP=10.10.10.10` to your .zshrc file. Keep in mind that .zshrc is only loaded when a new terminal starts. Either open a new terminal or manually reload it using `source $HOME/.zshrc`:

```bash

 export IP=10.10.10.10   

```
 <div style="margin: 1rem 0;"></div>
Now if you want to target the machine its as simple as:

```bash 

 $ sudo nmap -sC -sV  -p- $IP --open            
 Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-14 15:11 EST  
 Nmap scan report for 10.10.10.10 
 
```
 <div style="margin: 2rem 0;"></div>


## Speeding Up Enumeration with Custom Functions
Enumeration is one of the most time-consuming steps. Instead of configuring the options and running each tool manually every time, simply define a function in your .zshrc file. These functions assume you have already defined a target IP using the IP variable, as shown in the previous section.

Here’s a simple example for nmap:
```bash
 
 qnmap() {
  sudo nmap -sC -sV  -p- $IP --open   
  }
  
```
 <div style="margin: 1rem 0;"></div>
Here's another example, this time for feroxbuster:
```bash

 qferox() {
   feroxbuster --url http://$IP  -t 50 -E -B -g    
   }

```
## Expanding on the Basics
With the basics in place, we can expand on the ideas previously shown by using functions to define and manage targets, chain multiple enumeration commands and create organized output files.

### Creating a Function to Define the Target 
At this point, manually exporting the target IP works well, but it still requires editing your .zshrc configuration after every engagement. With this function, we can streamline this even further by dynamically setting the target IP on demand.
```bash

 setip() {
  sed -i '/^export IP=/d' ~/.zshrc
  echo "Target IP set to $IP"
  echo "export IP=$1" >> ~/.zshrc
  export IP=$1
  echo "Target set to $IP"
}

```
 <div style="margin: 1rem 0;"></div>
 Now you can run `setip 10.10.10.10` instead of modifying your.zshrc file.
 
```bash

 $ setip 10.10.10.10
  Target IP set to 10.10.10.10   
 $ echo $IP                   
  10.10.10.10

```




### Combining Enumeration Commands With Custom Functions
The real benefit of functions comes from chaining multiple commands together. Instead of manually launching several directory scans one after another, you can bundle them into a single function and view the results in one readable output file. Keep in mind, this function assumes you are on Kali Linux and have seclists installed in the default path, and may take 5+ minutes to complete.
```bash

 qdirb() {
   OUTPUT="dir-enum.txt"
   TEMP="dir-enum-temp.txt"

   echo "[+] Running directory enumeration against $IP"
   : > "$TEMP"

   echo "[+] Running raft-small-directories..."
   wfuzz -Z -o raw \   
     -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt \    
     --hc 404 -t 50 \
     http://$IP/FUZZ \
     | sed -r 's/^[0-9]+:\s*//' \
     | grep -Ev '^(Target:|Total requests|Processed Requests|Filtered Requests|Requests/sec|=+)' \
     >> "$TEMP"
   echo "[+] Done"

   echo "[+] Running big.txt..."
   wfuzz -Z -o raw \
     -w /usr/share/seclists/Discovery/Web-Content/big.txt \
     --hc 404 -t 50 \
     http://$IP/FUZZ \
     | sed -r 's/^[0-9]+:\s*//' \
     | grep -Ev '^(Target:|Total requests|Processed Requests|Filtered Requests|Requests/sec|=+)' \
     >> "$TEMP"
   echo "[+] Done"

   echo "[+] Running quickhits.txt..."
   wfuzz -Z -o raw \
     -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt \
     --hc 404 -t 50 \
     http://$IP/FUZZ \
     | sed -r 's/^[0-9]+:\s*//' \
     | grep -Ev '^(Target:|Total requests|Processed Requests|Filtered Requests|Requests/sec|=+)' \   
     >> "$TEMP"
   echo "[+] Done"

   # Deduplicate full result lines
   awk '!seen[$0]++' "$TEMP" > "$OUTPUT"
   rm "$TEMP"

   echo "[+] Enumeration complete — results saved to $OUTPUT"
}

```

## Avoiding Common Issues

### IP Address or Functions not Updating
If your function appears to be using an old IP Address or a new function is not being recognized, remember that .zshrc is only loaded when a new terminal starts. Either open a new terminal or manually reload it using `source $HOME/.zshrc`.

### Over Automating Too Early 
Automation is meant to save time doing redundant tasks and techniques you're already extremely familiar with. If you do not understand what you are automating, you can miss important details that ultimately slow you down during execution.



## Where to go From Here
At this point, you’ve seen how even simple functions can significantly reduce time spent during engagements. The goal is not to automate everything, but to eliminate repetitive decisions so you can focus on identifying attack vectors and actually finding vulnerabilities instead of repeatedly typing the same few commands. From here, these ideas can be extended much further to automate enumeration across multiple protocols with a single command.
