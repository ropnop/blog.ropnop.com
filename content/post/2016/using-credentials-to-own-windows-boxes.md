---
title: "Using Credentials to Own Windows Boxes - Part 1 (from Kali)"
slug: "using-credentials-to-own-windows-boxes"
author: "ropnop"
draft: true
date: 2016-04-16
summary: "Du'h...if you have admin creds you can own a box. But how many different ways can you do it? Here's a blog-ified version of my notes and my favorite methods"
toc: true
share_img: "/images/2016/04/password_inspector-2.png"
tags: ["kali", "windows", "impacket", "metasploit", "shell"]
series: ["Using Credentials to Own Windows Boxes"]
---

*Note: This is the first in what will hopefully be a multipart series about different ways to gain remote code execution on Windows machines. This first post is a quick braindump of different techniques from Kali. Future posts will explain the more subtle differences and how they actually work*

---

This probably doesn't sound like a very interesting blog post already. Literally all this post is going to be is me showing you different ways to log in to a Windows machine with admin credentials. But...it's not the fact that I'm using valid credentials and authenticating to Windows, it's all the different ways you can go about doing it.

Here's the situation: you are on an internal network and you've recovered somebody's valid domain credentials. I'm not going to cover how that happened here but there are plenty of ways you can get them (e.g. cracked a hash from responder or social engineered someone). Now what do you actually do with these credentials?

This actually happened to me on one of my first pentests. I cracked an intercepted hash and found myself in possession of a valid domain account. But I didn't know the best way to use these credentials. In all my previous CTF's and HackMe's I got shell access through exploitation. And now here I am wondering how to not "exploit" a box, but straight up log into it. I ended up using RDP and booted the legit user from their workstation. Not very stealthy....but I promise you I got better over the years.

So I decided to start compiling a cheat sheet for myself of every way I've ever popped a shell using credentials. It's just a text file on my laptop, but I figured other's could benefit from it. Whatever your post-exploitation method of choice is (lately I'm a big fan of Empire), these techniques are your first step in. 

For the purposes of demonstration, here's the Domain credentials we're assuming we know:

* **User**: jarrieta@cscou.lab
* **Pass**: nastyCutt3r

---

## Spray and Pray
The fist step after recovering credentials is to see where they are actually good. I'm a big fan of using msfconsole and its database features for storing network scans. Metasploit provides the rough and dirty "smb_login" module to test/bruteforce credentials across a variety of hosts. Supply our creds in the `SMBUSER` and `SMBPASS` options, then use `services -p 445 -R` to populate RHOSTS with every host with 445 open

```
msf > use auxiliary/scanner/smb/smb_login
msf auxiliary(smb_login) > set SMBDomain CSCOU
SMBDOMAIN => CSCOU
msf auxiliary(smb_login) > set SMBUser jarrieta
SMBUser => jarrieta
msf auxiliary(smb_login) > set SMBPass nastyCutt3r
SMBPass => nastyCutt3r
msf auxiliary(smb_login) > services -p 445 -R
msf auxiliary(smb_login) > run
```
Then run it and watch the results:
![Metasploit's SMB Login](/images/2016/04/smb_login.jpg)
The output shows the account is valid on three hosts. Metasploit also tells us that jarrieta is an Administrator on 10.9.122.5. That's huge because it means we can remotely execute code on that host.

*Note: Any succesfull logins will be stored in the msf database. View them with `creds`*

**CrackMapExec**. This is a fairly new tool that I've fallen in love with lately. It's written in Python and is extremely fast for testing credentials and launching attacks on a large number of hosts. You can get it from here: https://github.com/byt3bl33d3r/CrackMapExec

You can use CME to spray credentials across a network as well from the command line:
![CrackMapExec testing credentials](/images/2016/04/cme_logins.jpg)
If the login has admin rights, CME indicates it by saying "Pwn3d!" :)

---
## Make it rain shells
We now know where our compromised user is an administrator at 10.9.122.5. How many ways can we get a shell? 

*Note: I'm just gonna rapid-fire show commands here w/o really explaining how they work. Look for future blog posts digging further into these tools*

**Metasploit psexec.** The old classic. It's actually been updated to take advantage of PowerShell if it's present, but the underlying technique hasn't changed:

![Metasploit's psexec module](/images/2016/04/metasploit_psexec.jpg)
There's also `auxiliary/admin/smb/psexec_command` if you want to just run a single command. This module can also take a range of RHOSTS


**Winexe.** An old *nix tool to execute Windows commands remotely. Built in to Kali or available [here](https://sourceforge.net/projects/winexe/). You can execute a single command or drop right into a command prompt:
![Winexe on Kali](/images/2016/04/winexe_kali.png)

**psexec.py**. Part of the incredibly awesome [Impacket library](https://github.com/CoreSecurity/impacket). Seriously great work with these Python libraries and tools (and they're what CME is built on). The Kali version is a bit behind so I clone it to opt and install in a virtualenv.

![Impacket's psexec.py](/images/2016/04/psexec_imp.png)

**smbexec.py**. Another Impacket script. This one is a bit "stealthier" as it doesn't drop a binary on the target system. Commands and output are asynchronous:

![Impacket's smbexec.py](/images/2016/04/smbexec_imp.png)

**wmiexec.py**. Yet another awesome Impacket script (have I mentioned I like this project??). Under the hood this one uses Windows Management Instrumentation (WMI) to launch a semi-interactive shell.

![Impacket's wmiexec.py](/images/2016/04/wmiexec_imp.png)

**CrackMapExec**. You can also use CrackMapExec to execute commands on hosts by passing it the "-x" parameter. Since it's built on Impacket's libraries, it's basically doing the exact same thing as wmiexec.py, but let's you do it across a range of IPs:

![CrackMapExec -x options](/images/2016/04/crackmap_exec.png)


**Using Remote Desktop**. So this is kind of cheating since it's not really "from Kali", but sometimes it's your only option. You can RDP into the host and run a command from there. Like I mentioned earlier, this is what I did on my first pentest. I booted the actual user off here workstation. She then logged back in and booted me off. I had a command ready to go, then I logged back in, booted her off, and got a connect back Meterpreter before she logged back in and kicked me off. *Side note: I also had her email open in OWA and saw her send an email to IT saying she kept getting kicked off her machine. Not very stealthy at all....shame on you young ropnop*

You can use Impacket's `rdp_check` to see if you have RDP access, then use Kali's `rdesktop` to connect:
![rdp\_check and rdesktop to command prompt](/images/2016/04/rdpcheck_rdesktop.png)

---

## Other methods
There are, of course, many other things you can do with valid Windows credentials. These are just my go-to methods for getting a quick shell. Generally speaking, I rarely spend much time in the actual shell - I just use these methods to execute a post-exploitation toolkit, like Powershell Empire or a Meterpreter payload. These are just your "foot in the door".

Besides executing commands, you can RDP (as seen above), or mount SMB shares and download/upload files arbitrarily:

![smbclient browse files](/images/2016/04/smbclient_browse.png)


Next up I'm going to show ways to execute commands remotely and get shells via builtin Windows tools (you can't do everything from Kali...)

Did I miss anything? I'll keep this updated as I go. Feel free to comment and let me know your favorite method.

-ropnop