---
title: "Practical Usage of NTLM Hashes"
author: "ropnop"
date: 2016-06-05
summary: "I've shown all the different ways to own a Windows environment when you have a password - but having a hash is just as good! Don't bother cracking - PTH!"
toc: true
share_img: "/images/2016/06/you_shall_not_pth-1.jpg"
tags: ["pth", "mimikatz", "windows", "linux", "impacket", "crackmapexec"]
---

In my [last series](https://blog.ropnop.com/using-credentials-to-own-windows-boxes/), I discussed various ways of getting command execution in Windows environments when you've compromised a valid set of domain credentials.

In this post, I wanted to walk through some ways you can achieve the same results without ever actually needing a password. 

## Long live PTH
Pass-the-hash has been around a long time, and although Microsoft has taken steps to prevent the classic PTH attacks, [it still remains](http://www.harmj0y.net/blog/penetesting/pass-the-hash-is-dead-long-live-pass-the-hash/).

I'm not going to go into all the different ways you could recover a hash, but it's important to note the difference in certain types of hashes. In [Part 1](https://blog.ropnop.com/using-credentials-to-own-windows-boxes/), I talked briefly about recovering a domain account hash using Responder. The recovered password hash is in the format "NetNTLMv2", which basically means it's a "salted" NTLM hash. (I say salted because it's a little easier to understand, but really it's a hashed response to a challenge). This type of hash *can not* be used with PTH. If you've recovered one of these hashes, all you can really hope for is to crack it offline or try to capture it again and perform an SMB relay attack (a topic for another post).

The types of hashes you *can* use with PTH are NT or NTLM hashes. To get one of these hashes, you're probably gonna have to exploit a system through some other means and wind up with SYSTEM privs. Then you can dump local SAM hashes through Meterpreter, Empire, or some other tool. Mimikatz will also output the NT hashes of logged in users.

For this scenario, we'll assume I compromised a machine through some exploit, got an Empire agent, ran Mimikatz and recovered some NT hashes of valid domain users:

![Empire compromised hashes](/images/2016/06/empire_hashes.png)

We now have NT hashes for two domain users: kbryant, and jhoyer.

## Testing Logins with Hashes
CrackMapExec has become my go-to tool for quickly pentesting a Windows environment. I used it in an earlier post to test credentials across a network, but it also supports authenticating via PTH with NT hashes as well. To see if the "kbryant" hash can be used to authenticate anywhere on the network, use the `-H` option:

![CrackMapExec with a hash](/images/2016/06/crackmap_hashes.png)

It works and we authenticate as him and see that he's an admin on two different workstations.

Metasploit's `smb_login` can also be used with hashes to test credentials and see if a user is an Administrator. Metasploit requires the full NTLM hash, however, so you have to add the "blank" LM portion to the beginning: `aad3b435b51404eeaad3b435b51404ee`.

![Metasploit smb_login with hashes](/images/2016/06/metasploit_hashes.png)

## pth-toolkit and Impacket 
Pretty much all of the tools I outlined in [Part 1](https://blog.ropnop.com/using-credentials-to-own-windows-boxes/) can be used with NT hashes instead of passwords. 

The "pth" suite contains a bunch of programs that have been patched to support authenticating with patches:  
```pth-curl
pth-net
pth-rpcclient
pth-smbclient
pth-smbget
pth-sqsh
pth-winexe
pth-wmic
pth-wmis
```
More info at the Github page [here](https://github.com/byt3bl33d3r/pth-toolkit). *(sidenote: ha! i just noticed it's the same author as CrackMapExec. nice)*

I won't go into the tools again since they're the same, we're just using a Hash instead of a plaintext password now.

**pth-winexe**. The pth suite uses the format DOMAIN/user%hash:
![pth-winexe](/images/2016/06/pth-winexe.png)

**Impacket**. All the Impacket examples support hashes. If you don't want to include the blank LM portion, just prepend a leading colon:
![wmiexec hashes](/images/2016/06/wmiexec_hashes.png)

## Using Hashes with Windows
From within Windows, the two main tools to use with hashes are Impacket and Mimikatz.

I just found out recently that a researcher compiled all the Impacket examples to standalone Windows executables. If you can pull down the binaries onto your Windows system, you have all the amazing functionality of Impacket's examples from a Windows command prompt. Download the executables from here:

https://github.com/maaaaz/impacket-examples-windows

From my Windows attack box, I now have the same functionality as above:

![Windows wmiexec with hashes](/images/2016/06/windows_wmiexec.png)

Pretty slick.

## PTH with Mimikatz
If you don't know already, Mimikatz is so much more than just a tool to dump passwords from LSASS memory. It has PTH functionality builtin and can be used with a hash to essentially "runas" another process.

From within a command prompt (or PowerShell if you're using Invoke-Mimikatz), run the `sekurlsa::pth` module and specify the user, domain and NTLM hash. This will pop open another cmd prompt as if you just successfully did a "runas" with the kbryant user.

![Using mimikatz to PTH](/images/2016/06/mimikatz_pth.jpg)

We ran the `pth` module and a new command prompt opened up. We got a TGT for the "kbryant" user and can now run any Windows command as him.

## Coming Up
This was just a quick post showing that an NTLM hash is pretty much just as good as a password when it comes to security tools. If you compromise a hash, all the same tools and techniques are still valid and you can authenticate as that user to browse file shares and execute commands.

In my next post, I'm going to show how you can also do these techniques with compromised or forged kerberos tickets. Everyone hears about the dreaded "Golden Ticket" but I haven't really found much out there about the practical ways to use one if you have it. I'll dive into to generating a golden ticket and then using Linux and Windows tools to authenticate with it.

Lemme know if there's any tool or technique I missed or you want me to dive into more!

-ropnop
