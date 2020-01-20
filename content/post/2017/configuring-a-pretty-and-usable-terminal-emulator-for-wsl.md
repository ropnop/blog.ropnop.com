---
title: "Configuring a Pretty and Usable Terminal Emulator for WSL"
slug: "configuring-a-pretty-and-usable-terminal-emulator-for-wsl"
author: "ropnop"
draft: true
date: 2017-09-29
summary: "I'm a big fan of Bash on Windows (WSL), but was unable to find a good terminal emulator to use. In this post I talk about configuring Terminator for WSL"
toc: true
share_img: "/images/2017/09/terminator_windows-2.png"
tags: ["wsl", "bash", "zsh", "terminator"]
---

# Introduction
I've been using a Mac as my daily driver for work for the last few years. While there's nothing particularly special about MacOS that I love (in fact there's quite a bit I *don't* like), it's honestly been the terminal and the underlying Unix based operating system that keep me glued to it. With Homebrew, command line tools *just work*. Python and Node dev environments *just work*. And using [iTerm2](https://www.iterm2.com/) with [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh) is the best terminal experience I've ever had. I often feel like I just pay the premium for Mac hardware to have a reliable and easy to configure *Nix operating system.

But lately I've really been wanting to get off the Mac ecosystem and start using Windows 10 on my X1 Carbon as my daily machine. With the [Windows Subystem for Linux](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) (WSL) it's now possible to have a "native" Ubuntu command line on my Windows 10 machine to use for my CLI nerdiness. But the only thing holding me back was the lack of a nice terminal emulator (admittedly, I'm shallow and like pretty things).

This just wasn't going to cut it:
![Bash on Windows Default](/images/2017/09/bash_on_windows_default.png)

After much tinkering, I've ended up with what I feel is the most comfortable terminal experience I can get on Windows. It supports tabs, splits, mouse mode and has a pretty color scheme to boot:

![Terminator Windows](/images/2017/09/terminator_windows-1.png)

In this post, I'm going to quickly explain how I got it running and configured, and some of the other options I tried.

# First Attempts
I really just wanted the equivalent of iTerm2 in Windows. I wanted to utilize WSL (not Cygwin) and at a minimum needed:

* Pretty colors and fonts
* Tabs (non-negotiable)
* Working mouse support for scrolling and Vim/Tmux
* Tmux support and auto resizing
* Sane copy/paste

I think I tried every major Windows terminal app I could find. Each had their own drawbacks and I eventually gave up. Some of the ones I tried:

* [Cmder](https://conemu.github.io/)
 * Pros: Tab support. Portable. Works with cmd and PS nicely
 * Cons: Lacked mouse support in Tmux; resizing Windows was funky
* [ConEmu](https://conemu.github.io/)
 * Same as Cmder - not as pretty though by default
* [Wsltty](https://github.com/mintty/wsltty)
 * Pros: Window resizing and mouse worked great
 * Cons: No tabs!
* [Mintty](https://mintty.github.io/)
 * Same as Wsltty, just harder to configure initially
* [Hyper](https://hyper.is/)
 * Pros: Screenshots online made it look pretty. Apparently has nice plugins
 * Cons: Buggy as hell. Never got to properly work. Plugins based on NPM failed all the time
* [Babun](http://babun.github.io/)
 * Pros: Lots of features out of the box
 * Cons: Based on Cygwin, not WSL
* [Xshell](https://www.netsarang.com/products/xsh_overview.html)
 * Pros: ?? 
 * Cons: Didn't support WSL Bash. Not free 
* [MobaXTerm](http://mobaxterm.mobatek.net/)
 * Pros: A lot. Love this app for managing remote connections (e.g. RDP)
 * Cons: Not the best for local shells. Mouse/tmux support not working

The closest I got, and one that I used for a while was Cmder:

![Cmder Working](/images/2017/09/cmder_working.png)

Unfortunately, when I started using Tmux it became a problem. I could never get mouse mode to work (scrolling or selecting panes), and resizing windows was problematic. I'd end up with screens like this a lot:

![Cmder Broken](/images/2017/09/cmder_broken.png)

Not gonna cut it for me (though I still do use Cmder regularly for when I need to run Windows cmd.exe)

# Linux Terminal Emulators
What I realized in my search and multiple trials was there just *wasn't* a good Windows terminal emulator. When I was about to give up, I saw a post on Reddit about someone who got XFCE working on WSL Bash. That was way overkill for what I wanted to accomplish, but reading through the post I learned/realized that if I had an X Server running on Windows, I could use GUI Linux terminal emulators "natively" on Windows! That opened up a ton of possibilities, and one of my favorite Linux terminals, [Terminator](https://gnometerminator.blogspot.com/), was now a possibility!

## Installing an X Server
To run an X Window application, I needed to have an X Server installed and running on my Windows 10 machine. After researching, it seemed the two most popular options are:

* [VcXsrv](https://sourceforge.net/projects/vcxsrv/)
* [Xming](https://sourceforge.net/projects/xming/)

I went with VcXSrv since it looked like it was more actively maintained, but I tried both and they work the same.

After installing, VcXsrv creates a desktop shortcut to start the server in multi-window mode through the following command:

```b
"C:\Program Files\VcXsrv\vcxsrv.exe" :0 -ac -terminate -lesspointer -multiwindow -clipboard -wgl -dpi auto
```
A taskbar icon shows it's running, and we can verify by looking at netstat:

```b
C:\Windows\system32>netstat -abno|findstr 6000
  TCP    0.0.0.0:6000           0.0.0.0:0              LISTENING       6216
```
It's important to note that VcXsrv is listening on all interfaces and requests a blanket firewall exception for private networks. AFAIK there is no way to only force it to listen/accept connections from localhost only, so I disallowed the firewall exception request and configured a custom rule to only allow traffic from `127.0.0.1`

![Firewall Rule](/images/2017/09/VcXSrv-Local-Allow-Properties-2017-09-28-18-26-13.png)

## Configuring Terminator
Once VcXsrv was installed and configured to allow access from `127.0.0.1`, the next step was to install Terminator on WSL Bash:

```bash
$ sudo apt-get install terminator
```

If I didn't want to use terminator, any other terminal emulator should work, including Gnome Terminal (which Terminator is based on), Urxvt, or xterm.

After it installed, all that was left was to try launching it by specifying the X Display to connect to (`:0`)

```bash
$ DISPLAY=:0 terminator &
```

And a nice Terminator windows popped up :)

![Launching Terminator](/images/2017/09/launching_terminator.png)

### Installing Zsh
The next step I did was install Zsh with [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh). Installation is straightfoward:

```b
sudo apt-get install curl wget git zsh
curl -L https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh | bash
```
I set the theme "ys" in `.zshrc`

The only "gotcha" about using Bash in WSL is it will always run Bash instead of Zsh. To get around that, I add this to the end of my `.bashrc` which will launch `zsh` instead when it starts up:

```bash
if [ -t 1 ]; then
  exec zsh
fi
```

### Terminator Colorscheme
The next thing I chose to do was change the default Terminator colorscheme to Solarized Dark. The easiest way to do this IMO, is to use the awesome node package [base16-builder](https://github.com/base16-builder/base16-builder)

```b
sudo apt-get install nodejs-legacy
sudo npm install --global base16-builder
mkdir -p .config/terminator
base16-builder -s solarized -t terminator -b dark > .config/terminator/config
```

### Dircolors
It was looking good so far, but the dircolors were still awful:

![Bad dircolors](/images/2017/09/bad_dircolors-1.png)

I found [Solarized dircolors](https://github.com/seebi/dircolors-solarized) on Github and downloaded them to `.dir_colors`

```b
wget https://raw.githubusercontent.com/seebi/dircolors-solarized/master/dircolors.256dark
mv dircolors.256dark .dir_colors
```
Then lastly, added this to my `.zshrc` to eval the Solarized dircolors on startup:

```bash
if [ -f ~/.dir_colors ]; then
  eval `dircolors ~/.dir_colors`
fi
```
Finally I had a pretty enough terminal to my liking. Still no iTerm2, but pretty damn close IMO

# Launching Terminator Directly
The final hurdle was figuring out a way to launch Terminator directly without having to open a Bash window first and then typing `DISPLAY=:0 terminator &`

From the command line, I could launch bash with the `-c` parameter and it would work:

```b
C:\Users\ronnie>bash -c -l "DISPLAY=:0 terminator &"
```

However, this still required me to open a command window. After a lot of trial and error and research on StackOverflow, I figured out how to launch a hidden command window using the WShell Object in VBS. The final script I used is here:

{{<gist ropnop 10800fb5066bd5144d9aaad55a8a4d18>}}

With that script, I was able to create a Desktop shortcut to `wscript.exe` and execute it with the following command:

```b
C:\Windows\System32\wscript.exe C:\Users\ronnie\startTerminator.vbs
```

![Terminator Shortcut](/images/2017/09/terminator_shortcut.png)

The last step was to find a pretty Terminator [Icon file](https://goo.gl/images/kvLttb) for the shortcut and add it to my taskbar for quick launching.

One final note is the "Start In" option. It's impossible to have Termiator start in my Linux home directory through this method since that path is not "known" to Windows. To get around it, I added this to my `.zshrc` so it CD's to my home directory on startup:

```bash
if [ -t 1 ]; then
  cd ~
fi
```
Hacky, but it works.

# Conclusions
I've been using Terminator for WSL for a while now and am loving it. It's the best terminal experience I've been able to get on Windows so far. As long as I have VcXsrv running I've had no issues with launching it and it runs very smoothly. Terminator is very configurable, and I've configured and changed some of the keyboard shortcuts to suit my liking. Overall I love having the splits/panes/tabs and when I'm SSH'd into multiple boxes through WSL it's amazing. My workflow now is generally running VS Code on my Windows, with multiple Terminator panes open to `/mnt/c/projects/whatever` and being SSH'd into my lab. Love it.

![Terminator Windows](/images/2017/09/terminator_fullscreen.png)


Hope this helps someone! Let me know if you have a different/better solution!

-ropnop
