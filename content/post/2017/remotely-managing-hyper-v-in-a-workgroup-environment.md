---
title: "Remotely Managing Hyper-V in a Workgroup Environment"
slug: "remotely-managing-hyper-v-in-a-workgroup-environment"
author: "ropnop"
date: 2017-08-01
summary: "After lots of mucking around, this is the bare minimum configuration I found to successfully connect to and mange Hyper-V in a non-domain network"
toc: true
share_img: "/images/2017/08/hyper-v-status-1.png"
tags: ["windows","winrm","powershell","hyperv","credssp"]
---

# Background
A few weekends ago, I decided (because apparently I'm a masochist) that I was tired of the free version of ESXi running my home lab and decided to give Microsoft's [Hyper-V Server 2016](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/hyper-v-server-2016) a try. 

I liked the idea that it was light weight, accessible with PowerShell, and there were [lots](https://github.com/AutomatedLab/AutomatedLab) of [cool](https://github.com/PlagueHO/LabBuilder) Github projects that made deploying test AD domains automatic.

I had played around with Hyper-V on my Windows laptop, and liked the GUI Hyper-V manager as well. If I could remotely manage my lab with that and PowerShell, I'd be happy.

Installing Hyper-V server from the ISO was easy, but little did I know I'd burn several days just trying to connect to it.

I followed the steps in these docs, but still never got it working:

* https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/manage/remotely-manage-hyper-v-hosts

* http://pc-addicts.com/remotely-manage-hyper-v-server-2012-core/

The crux of my problem is that Hyper-V Server is really intended to be run in an AD Domain environment. Remotely managing via PowerShell or the Hyper-V Manager I'm sure works seamlessly if AD is behind it handling authentication and authorization.

I don't have an AD Domain set up in my apartment, and had little desire to spin up a DC and manage a domain for personal use. So I set about trying to configure Hyper-V to play nice with a WORKGROUP. What I wanted to do was be able to remotely connect to and manage the server from my main Windows workstation. I also wanted to do the bare minimum amount of tweaking to my main workstation (i.e. not doing anything blatantly insecure like disabling firewalls, trusting wildcards, etc)

Below are the commands I found that work to get remote connections working. 


# Configuring the Server
After initial installation, and setting an Administrator password, you're presented with this minimalist screen:

![HyperV Main Screen](/images/2017/08/main_screen.png)

There's a few things that need to be configured from here.

1. In "Configure Remote Management", make sure Remote Management is enabled and change the setting to allow the server to be pinged

2. Enable Remote Desktop so you can remotely manage the server over RDP

3. Configuring a name. I chose `hyperv-server`. This requires a reboot.

I also created a host entry for `hyperv-server.home.local` in my local DNS server so  I could access it by hostname. Manually editing the `etc\hosts` file would work as well.

After these settings, I was able to RDP into the box by its hostname and using the credentials for `HYPERV-SERVER\Administrator`.

## Server PowerShell Configuration
Behind the configuration menu is a command prompt, which lets you drop into PowerShell. From here, you need to configure the server to allow PS-Remoting and to allow CredSSP authentication to the server. Use the following commands:

```powershell
Enable-PSRemoting
Enable-WSManCredSSP -Role server
```

![Enabling PowerShell remoting](/images/2017/08/enable_remoting.png)

Once that's enabled, you're done configuring the server. Now came the tricky part of getting my client to communicate with it.

## Client Configuration
I would be remotely managing the Hyper-V Server from my Windows 10 laptop. The first thing is to make sure that the connection profile is set to "Private" so that Windows Firewall allows the traffic (i.e. it won't work over a public network)

```powershell

Get-NetAdapter|Get-NetConnectionProfile

Set-NetConnectionProfile -InterfaceAlias Ethernet -NetworkCategory Private
```

The next step is to configure the client that it's okay to connect to the server by adding the server as a trusted host for the [WS-Management Protocol](https://msdn.microsoft.com/en-us/library/aa384470(v=vs.85).aspx). 

To add the entry, you have to enable WinRM on the client. I did not want to enable WinRM on my laptop, but fortunately I discovered that stopping the service after adding the entry works. To add the entry:

```powershell
$ Start-Service -Name winrm
$ Set-Item WSMan:\localhost\Client\TrustedHosts -Value "hyperv-server.home.local
$ Stop-Service -Name winrm
```
*This should work with an IP address instead of hostname, but I didn't test. YMMV*

Now test out that you can authenticate to and remotely run commands from your workstation by testing `Invoke-Command`:

```powershell
Invoke-Command {whoami} -ComputerName hyperv-server.home.local -Credential HYPERV-SERVER\Administrator
```
It will prompt you for the local Administrator password and then remotely execute the `hostname` command:

![WinRM and hostname](/images/2017/08/invoke_command.png)

If this works, you can now securely communicate between client and server and can finally start remotely managing Hyper-V.

## Configuring Hyper-V Manager
If you haven't already, make sure to enable `Hyper-V Management Tools` in Windows Features to get the Hyper-V Manager GUI. You don't have to actually enable the Hyper-V Platform (I don't since I run Virtualbox locally).

![Enable Hyper-V Tools](/images/2017/08/hyper-v_tools.png)

Open up Hyper-V Manager, right click and select "Connect to Server". Enter the IP address or hostname of the Hyper-V server, and choose "Connect as another user", entering the local Administrator credentials

![Select Computer](/images/2017/08/hyperv_connect_to.png)

![Select User](/images/2017/08/hyeprv_connect_as.png)

Now...you'd think this would totally work. Especially since we've already proved that WinRM and PS-Remoting work just fine. But hit 'OK' and this error pops up:

![Hyper-V Error](/images/2017/08/hyperv-error.png)

I have no idea why this is failing. This had me at a loss until I found a workaround.

## Authorization Workaround
To get around the authorization error, you have to manually add the Hyper-V Server credential into memory using `cmdkey`. Exit out of Hyper-V server and enter the following at a command prompt on the client:

```batch
cmdkey /add:HYPERV-SERVER /user:Administrator /pass:<REDACTED>
```
Now re-open Hyper-V Server and connect to the remote server again, but don't bother specifying another user since the creds we need are already in memory:

![HyperV Success](/images/2017/08/hyper-v-no-user.png)

This finally works! You can connect and remotely manage the server:

![HyperV Status](/images/2017/08/hyper-v-status.png)

As long as the credential remains in memory, you can remotely manage the server. Another possible workaround is to start Hyper-V Manager with `runas` and the correct local account:

```batch
runas /user:HYPERV-SERVER\Administrator /netonly "mmc virtmgmt.msc"
```
You could create a shortcut to run this and be good to go.

# tl;dr
**On Hyper-V Server**

* Enable Remote Management
* Enable Remote Desktop
* Set Computername
* Enable WSMan and PS-Remoting:
```powershell
Enable-PSRemoting
Enable-WSManCredSSP -Role server
```

**On Managing Client**

* Configure hosts or DNS entry for Hyper-V Server
* Ensure network profile is "Private":
```powershell
Set-NetConnectionProfile -InterfaceAlias Ethernet -NetworkCategory Private
```
* Temporarily start WinRM and add Hyper-V Server as Trusted Host:
```powershell
Start-Service -Name winrm
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "hyperv-server.home.local"
Stop-Service -Name winrm
```
* Enable Hyper-V Management Tools in Windows Features
* Add Hyper-V Administrator credential with cmdkey:
```batch
cmdkey /add:HYPERV-SERVER /user:Administrator /pass:<REDACTED>
```
* Launch Hyper-V Manager and connect to server (don't specify user)
  * or, launch Hyper-V Manager with `runas`:
```batch
runas /user:HYPERV-SERVER\Administrator /netonly "mmc virtmgmt.msc"
```
## Summary
After lots of mucking around, this is the bare-minimum working configuration I've found to remotely manage a Hyper-V server in a non-domain environment.

I'm still not 100% certain why Hyper-V Manager does not work with supplied credentials and the `cmdkey` or `runas` workaround is necessary. If anyone can shed some light on that, I would appreciate it!

Hope this helps someone out there
-ropnop


