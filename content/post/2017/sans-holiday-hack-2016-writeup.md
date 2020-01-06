---
title: "Sans Holiday Hack 2016 - Writeup"
slug: "sans-holiday-hack-2016-writeup"
author: "ropnop"
date: 2017-01-05
toc: true
share_img: "/images/2017/01/game_background.png"
tags: ["sans", "android", "burp", "writeup", "holidayhack"]
series: ["SANS Holiday Hack Write-ups"]
---

---
After my last report for work went out the door and my company entered its end-of-year shutdown period, I found myself at my parents house for several days for the holidays, relaxed and with nothing to do. I saw some people on Twitter talking about the SANS Holiday Hack Challenge, and decided I would finally give it a try.

I started on Christmas Eve and after several days of borderline dangerous obsessive completion-compulsion, I had solved all the challenges. All challenges were amazingly well done, and unlike some other CTFs I've participated in, they were all based on "real-world" examples and techniques which gave me a chance to practice and hone my skills and learn some new ones.

The challenge starts here:

https://holidayhackchallenge.com/2016/

So without further ado, here is my write up on solving the SANS Holiday Hack Challenge 2016.


*Note: Sorry for the long writeup, but I wanted to try to fully capture my thought process on each of these challenges, and discuss the failures and rabbit holes I went down. I'm not a huge fan of solution writeups that just say "here's the solution, all it takes is this oneliner". They completely ignore all the buildup and discovery, which are super important parts for pentesters. We're not infallible and we definitely don't get the solution immediately and in one try.*

## Part 1: A Most Curious Business Card
I started the the hack challenge by entering the [game world](https://quest2016.holidayhackchallenge.com/) and finding my first clue, a business card from Santa:

![Santa's Business Card](/images/2017/01/santas_business_card.png)

It pointed to his [Twitter](https://twitter.com/SantaWClaus) and [Instagram](https://www.instagram.com/santawclaus/) accounts.

## Twitter Message
> 1) What is the secret message in Santa's tweets?

When I saw his Twitter account, my heart sank a little since I hate crypto and stego challenges. But I wasn't going to give up on step 1, so I figured the first step would be to grab all his tweets and run some frequency analysis and maybe some sort of code will come out of it.

I had heard of [Tweepy](https://github.com/tweepy/tweepy) before but hadn't used it. I knew it had to be fairly straightforward to download a user's tweets. I found this [great walkthrough](https://marcobonzanini.com/2015/03/02/mining-twitter-data-with-python-part-1/) about data mining in Twitter using Tweepy. Then I actually found a public [Gist](https://gist.github.com/yanofsky/5436496) which does exactly what I wanted. It needs Twitter API credentials, so I created a new Twitter app on https://apps.twitter.com/ to get an API key, API secret, and access token.

Once I had all that, I modified the script to dump tweets from @santawclaus (not @santawclause as I first tried - really confused why it wasn't working):

```bash
$ python tweet_dumper.py
getting tweets before 798175247283392512
...350 tweets downloaded so far
getting tweets before 798175028978352128
...350 tweets downloaded so far
```

The script downloaded the tweets to `santawslcaus_tweets.csv`. The CSV format is `id,created_at,text` so I used awk to ignore the first two fields and dump the text:

![Tweet Content](/images/2016/12/tweet_content_B.png)

A "B"? Sweet it's ASCII art! No crypto or stego needed :D Scrolling down through the tweet content, it spells out: **BUGBOUNTY**

## Instagram Clues
> 2) What is inside the ZIP file distributed by Santa's team?

I was supposed to find a ZIP file, but I didn't know where it was yet. I hadn't looked at Santa's Instagram account. There was one picture in particular that probably contained clues: https://www.instagram.com/p/BNpA2kEBF85/

I don't think I ever realized until doing this challenge that Instagram doesn't give you an easy way to download images locally. I used a free [online tool](https://downloadgram.com/) to grab a local copy of the image:

![Santa's Instagram Pic](/images/2016/12/santa_insta_pic.jpg)

Zooming in and looking at Santa's laptop screen, I saw the tail end of a command that lists a ZIP file: `SantaGram_v4.2.zip`. On the right side of the picture, tucked into Violent Python (great book, btw), was the top of an Nmap scan report for www.northpolewonderland.com. 

Putting the two together, I ventured a guess and tried to download the Zip file from the URL:

```bash
$ wget www.northpolewonderland.com/SantaGram_v4.2.zip
--2016-12-29 11:16:37--  http://www.northpolewonderland.com/SantaGram_v4.2.zip
Resolving www.northpolewonderland.com... 130.211.124.143
Connecting to www.northpolewonderland.com|130.211.124.143|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1963026 (1.9M) [application/zip]
Saving to: ‘SantaGram_v4.2.zip’

SantaGram_v4.2.zip.1 100%[======================>]   1.87M   848KB/s    in 2.3s

2016-12-29 11:16:40 (848 KB/s) - ‘SantaGram_v4.2.zip’ saved [1963026/1963026]
```

Great - I found the Zip file! I can see that inside the Zip is an APK file, but trying to inflate it prompts for a password. I tried the secret message from the tweets ("BUGBOUNTY") but no luck. Then I tried it lower case and it worked!

```bash
$ unzip -l SantaGram_v4.2.zip
Archive:  SantaGram_v4.2.zip
  Length     Date   Time    Name
 --------    ----   ----    ----
  2257390  12-09-16 07:47   SantaGram_4.2.apk
 --------                   -------
  2257390                   1 file

$ unzip SantaGram_v4.2.zip -d android/
Archive:  SantaGram_v4.2.zip
[SantaGram_v4.2.zip] SantaGram_4.2.apk password:
  inflating: android/SantaGram_4.2.apk
```

I now have the first two answers:
>1) What is the secret message in Santa's tweets?

**BUGBOUNTY**
>2) What is inside the ZIP file distributed by Santa's team?

**An APK file: SantaGram_4.2.apk**

## Part 2: Awesome Package Konveyance
> 3) What username and password are embedded in the APK file?
>
>4) What is the name of the audible component (audio file) in the SantaGram APK file?

Time for some Android analysis! First, I extracted the APK contents to look for the audible component. Although an APK is just a ZIP file, simply unzipping it won't work as expected. A lot of the files, like the manifest, will still be encoded and unreadable. Instead, I used `apktool` to extract and decompile all the contents:

```bash
$ apktool d SantaGram_4.2.apk
I: Using Apktool 2.2.0 on SantaGram_4.2.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /Users/rflather/Library/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

Next, I used `find` to list all the static files inside the newly created SantaGram_4.2 directory. I didn't care about `.smali` or `.xml` files right now, so I ignored them:

```bash
$ find SantaGram_4.2 -type f ! -name '*.smali' ! -name '*.xml'
```

It was mostly image files (.png), but scrolling through the list I finally saw the audio file: `SantaGram_4.2/res/raw/discombobulatedaudio1.mp3`

### Decompiling SantaGram
To find the embedded username and password I could search through the smali code since strings should be preserved, but I'd much rather look at Java. The in-game elves gave some great hints for this part, including using `jadx`, but my favorite tool to use when looking at Android apps is [ByteCodeViewer](https://github.com/Konloch/bytecode-viewer). I downloaded the latest release and launch it from the command line, specifying using additional memory:

```bash
$ java -jar -Xmx2048m ~/tools/bytecodeviewer/BytecodeViewer\ 2.9.8.jar
```

When it opened, I simply dropped the entire APK file in and it decompiled everything. The Search feature was my friend here. I just searched for the string "password" and clicked through the files until I found what looked like some hard coded credentials for something related to Analytics:

![Analytics Credentials](/images/2016/12/bytecodeviewer_passwod.png)

Now I've answered the next 2 questions:
>3) What username and password are embedded in the APK file?

**guest : busyreindeer78**

> 4) What is the name of the audible component (audio file) in the SantaGram APK file?

**discombobulatedaudio1.mp3**

## Part 3: A Fresh-Baked Holiday Pi
For Part 3, I had to go back into the game world and find all the pieces for the Cranberry Pi. This took me wayyyy longer than I'd care to admit and I almost gave up since I just wanted to get back to "hacking" :)

*(hint: there's a secret room behind a fireplace)*

Once I had all the pieces, the elf at the beginning (Holly Evergreen) gave me a download link to the SD card image: https://www.northpolewonderland.com/cranbian.img.zip

One of the other elves gave a huge hint to a great blog post about [mounting Raspberry Pi images](https://pen-testing.sans.org/blog/2016/12/07/mount-a-raspberry-pi-file-system-image). I followed this guide and got the image mounted in a Kali system.

First, I used `fdisk -l` on the unzipped .img file to see what offset the filesystems were. There was a Linux filesystem at offset 137216. Since the sector size is 512 bytes, the true offset is at 70254592 bytes (512*137216). 

Knowing that, I mounted the filesystem to a folder called `mnt` with the following command:

```bash
root@kali:~/sanshh# mount -v -o offset=70254592 -t ext4 cranbian-jessie.img mnt/
mount: /dev/loop0 mounted on /root/sanshh/mnt.
```

Once it was mounted, I had full access to the filesystem under `mnt/`:

```bash
root@kali:~/sanshh# ls mnt/
bin  boot  dev  etc  home  lib  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

Since I wanted the password to the `cranpi` account, I pull it out of `/etc/shadow` from the mounted file system:

```bash
root@kali:~/sanshh# cat mnt/etc/shadow |grep cranpi
cranpi:$6$2AXLbEoG$zZlWSwrUSD02cm8ncL6pmaYY/39DUai3OGfnBbDNjtx2G99qKbhnidxinanEhahBINm/2YyjFihxg7tgc343b0:17140:0:99999:7:::
```

The prefix on the hash ("$6") means the format is sha512crypt. Since cracking on a VM is slow, I copied the entire hash back to my Mac and used `hashcat`. It's entirely possible to use `john`, but I prefer `hashcat` since it takes advantage of my MBP's GPU. 

Looking at the [hashcat wiki](https://hashcat.net/wiki/doku.php?id=hashcat) I see that sha512crypt is mode 1800. One of the elves gave a hint in-game about using the wordlist "rockyou.txt", which is included in the awesome Git repo [SecLists](https://github.com/danielmiessler/SecLists). So putting it all together:

![Using Hashcat](/images/2016/12/hashcat_cracking.png)

After about 2 minutes, my MBP spit out the cracked hash:

```bash
$6$2AXLbEoG$zZlWSwrUSD02cm8ncL6pmaYY/39DUai3OGfnBbDNjtx2G99qKbhnidxinanEhahBINm/2YyjFihxg7tgc343b0:yummycookies
```

So the local account on the CranPi is:

**cranpi : yummycookies**

In game, I told the password to Holly Evergreen and she told me I now had access to the Cranberry Pi terminals.

## Accessing the Terminals

Armed with the Cranberry Pi, I now had access to each terminal and could try to get the passwords needed to open the doors. These terminal "mini-games" were really fun and well-designed. I'll outline how I got the past each one.

### Elf House 2 Terminal - Peacoats and PCAPs
The first terminal was in the back of Elf House #2. The banner told me how to open the door:

![Elf House 2  - Banner](/images/2016/12/elfhouse2_banner.png)

Easy enough...but there's immediately a problem. I don't have permissions to read the file:

![Elf House 2 - Permissions](/images/2016/12/elfhouse_2_permissions.png)

So I might need to elevate privileges?

In [my talk](https://blog.ropnop.com/abusing-linux-trust-relationships-thotcon-slides/) at Thotcon and Derbycon this year, I talked about my favorite way to elevate privs - using misconfigured sudo commands. Running `sudo -l` on this terminal showed me some interesting commands:

![Elf House 2 - Sudo](/images/2016/12/elfhouse2_sudo.png)

So I could run `tcpdump` and `strings` as the user `itchy` without requiring a password! My first thought was that I could use `tcpdump` to run a command with the `-z` option and get a shell as `itchy`. There was a good discussion on this on SecLists [here](http://seclists.org/tcpdump/2010/q3/68). Unfortunately I realized that I would need to capture packets in order to run the command, and the user `itchy` doesn't have permissions to capture on any interface, so I abandoned that idea and decided to just focus on the two commands I could run. 

Unfortunately, `tcpdump` isn't really the best tool to examine pcap files (`tshark` is what I really wanted), but I could do some basic investigation using builtin Linux tools. There's a great SANS whitepaper on doing just that [here](https://www.sans.org/reading-room/whitepapers/protocols/analyzing-network-traffic-basic-linux-tools-34037).

First I scraped out the endpoints:

![Elf House 2 - Endpoints](/images/2016/12/elfhouse2_endpoints.png)

Seeing that there's HTTP traffic, I then pulled out the HTTP requests:

![Elf House 2 - HTTP](/images/2016/12/elfhouse2_http.png)

So there were two HTTP requests, one for `firsthalf.html` and another for `secondhalf.bin`. Since it's plaintext HTTP, I could use the `strings` command to pull out the HTML for the first half:

![Elf House 2 - Strings 1](/images/2016/12/elfhouse2_strings.png)

![Elf House 2 - Strings Output](/images/2016/12/elfhouse2_part1.png)

So Part 1 was "**santasli**".

Part 2 took me a long time to figure out. I knew I probably had to carve out `secondhalf.bin` to examine it. I looked around and realized there's no real easy way to carve out a file using only `tcpdump`. I found a Perl script [here](http://forensicscontest.com/contest01/Finalists/Kristinn_Guojonsson/pcapcat) which can read PCAP files and carve out files, but even though Perl was installed, I couldn't run it as `itchy` to actually read the file.

Playing around, I realized that I could use `tcpdump` to write the contents of `out.pcap` to another file, and that file would have global read permissions:

![Elf House 2 - tcpdump read/write](/images/2016/12/elfhouse2_read_write.png)

Now I could run the Perl script as the `scratchy` user and still access the file! 

I copy/pasted to code, but when I tried to run it, none of the required Perl modules (like Net::Pcap) were installed - d'oh! Scratch that idea.

I got pretty frustrated and thought it was dumb I couldn't use tshark or wireshark, so I started trying to see if I could pull the entire pcap file off to my local machine. The only real way I could do that was to base64 encode it and copy/paste from the terminal. The file is over 1 Mb and translated to about 19k lines of b64 encoded content. That seemed like big hassle and a lot of copy/paste so I revisited the two commands I could use, knowing I had to be missing something obvious.

Reading up on what `strings` could do, I stumbled across this SANS blogpost called [Strings are wonderful things](https://digital-forensics.sans.org/blog/2009/05/15/strings-strings-are-wonderful-things). Reading it, it dawned on me I never tried looking for unicode strings on the pcap (strings defaults to ASCII). I ran the command from the blog and:

![Elf House 2 - Strings 2](/images/2016/12/elfhouse2_strings2.png)

Ugh...I felt dumb. So simple. Lesson to self: always make sure to look for both ASCII and Unicode strings.

Combining the two parts, I had the password for the first console:

**santaslittlehelper**

### Workshop Terminal 1 - Gone Spelunking
The next terminal I tried was in Santa's Workshop, next to the reindeer pen. The banner told me what's needed:

![Wumpus - Banner](/images/2016/12/wumpus_banner.png)

There was a file called `wumpus` in my home directory. The `file` command was not present, but I could run `head` to see the magic bytes and know that this was an ELF binary:

![Wumpus - ELF](/images/2016/12/wumpus_head_elf-1.png)

When running the binary and trying to view the instructions, it told me that the instructions file has "disappeared":

![Wumpus - No Instructions](/images/2016/12/wumpus_instructions_gone.png)

Figuring the password was somewhere in the binary, I ran `strings` but didn't see anything that looked like a password. However, I did see some strings related to the instructions:

![Wumpus - strings](/images/2016/12/wumpus_strings_instructions.png)

Just by looking at the strings, I could kind of guess that the binary is trying to open `wump.info` with `/usr/bin/less`. The `PAGER` string was probably an environment variable being read too.

I created a file called `wump.info` in the same directory and ran the binary again, getting a different error:

![Wumpus - No Less](/images/2016/12/wumpus_less_not_found.png)

So it was trying to run `/usr/bin/less` through `sh` and `less` doesn't exist. What about that `PAGER` environment variable?

![Wumpus - PAGER](/images/2016/12/wumpus_id_exec.png)

Interesting...I found a command injection vuln in Wumpus! But wait...it's still running as the `elf` user. D'oh! It's not a `setuid` or `setgid` binary! So I found a way to run commands as myself. That's not gonna do me any good. Back to the drawing board!

Unfortunately `gdb` is not installed on the console, so I can't run the binary in a debugger. But I could do some static analysis on it. `readelf` and `objdump` are installed, so I decided to take a closer look at the binary. Fortunately the symbols haven't been stripped, so I used `objdump` to look at symbols in the `.text` section (where code would be):

![Wumpus - objdump symobls](/images/2016/12/wumpus_objdump_text.png)

I saw the `instructions` function, which is probably where I found that useless command injection vuln, and a potentially interesting function called `kill_wump`.

I assumed the point of the game was to kill the wumpus, so I decided to take a closer look at that function. I disassembled the binary to a text file using:
```bash
elf@641574f9daa5:~$ objdump -D wumpus > wumpdump
```
I wanted to use `vim` to search and explore, but quickly realized there was no text editors installed in the console, so I used `more` instead to look at the `kill_wump` function:

![Wumpus - kill_wump](/images/2016/12/wumpus_objdump_killwump.png)

I'm still pretty noobish when it comes to assembly, but what really jumped out at me was all the crazy `mov` commands before calling `puts`. It seemed to me maybe it was doing some sort of decryption/deobfuscation, which would explain why I didn't see the password in the strings output.

At this point, I knew I needed to call the `kill_wump` function, and also that I didn't want to actually "play" the game to get there. If I had `gdb` on this console it would be simple. Set a breakpoint on `main` and then just `jump kill_wump`. But how to jump to that function without a debugger? I'd have to modify the binary to call that function.

After looking around on the internet, I found a great resource about [modifying ELF binaries and changing callq instructions](https://www.pacificsimplicity.ca/blog/modifying-linux-elf-binaries-changing-callq-addresses).

At this point, I knew that the binary probably called the `instructions` function when it started (or when I pressed 'y'), and that I wanted it to instead call the `kill_wump` function. Looking through the disassembly of `main`, I saw the call to `instructions` at offset `40100e`:

![Wumpus - callq instructions](/images/2016/12/wumpus_callq_instructions-1.png)

I wanted to change the address it calls from `instructions` to `kill_wump`. The offset of `kill_wump` I got from the symbols table above: `402644`.

As the link I found pointed out, and what I didn't know, was that the `callq` operation jumps to a location relative to the location of the *next* instruction. I saw in the assembly that the next instruction after the `callq` is located at `401013`, so the difference between that and `kill_wump` is: 
402644 - 401013 = **1631**, or `31 16 00 00` in little endian.

The `callq` opcode (`e8`) wouldn't change, but the address needed to. So I needed to change the hex instruction on that line from `e8 8d 14 00 00` to `e8 31 16 00 00`. This would cause the binary to jump to the start of `kill_wump` instead of the start of `instructions`.

Knowing what I needed to do, I went to modify the binary and unfortunately realized I didn't have any hex editing tools (or even a text editor like vim, for that matter) on the console. Ugh, why is nothing ever easy?!? Fortunately, I love "living off the land" and working with what I have, and noticed that perl is installed, as well as text manipulation tools like `sed`.

So to modify the binary, I needed to dump it to hex, use `sed` to replace the instruction I needed changed, and then decode the hex back into binary. My perl knowledge is non-existent, so I turned to a pentester's best friend (Stackoverflow) and [found out how](http://stackoverflow.com/questions/2614764/how-to-create-a-hex-dump-of-file-containing-only-the-hex-characters-without-spac) to encode/decode files to/from hex with perl.

Putting it all together, I used a combination of perl and sed to modify the hex opcodes in the binary. Instead of calling the `instructions` function, it would now call the `kill_wump` function and *hopefully* spit out the password:

![Wumpus - modify and run](/images/2016/12/wumpus_modify_hex_perl-1.png)

Jackpot! The game now jumped ahead to the end instead of asking me if I want instructions, and I had recovered the passphrase from the wumpus:

**WUMPUS IS MISUNDERSTOOD**

Phew! Probably would have been easier to just play the game (or yank the binary off the console and use a debugger), but this way was more fun and I learned a ton :)

### Workshop Terminal 2 - The One Who Knocks
After spending a half day on Wumpus, it was nice to see a new challenge that seemed pretty straight forward:

![Deepdir - Banner](/images/2016/12/deepdir_banner.png)

Seemed simple enough. Often times on pentests when I gain access to someone's home directory, I like to spit out a list of all files to see if anything jumps out at me as maybe containing passwords or other sensitive information. I use the surprisingly powerful `find` command. The command will recursively search for everything under a directory. To see what files were under my current directory (using the `-type f` option to only display files, not directories):

![Deepdir - find files](/images/2016/12/deepdir_find.png)

Ah there it is! I could try using `cat` on it, but all those special characters would be a pain to escape on the command line. Instead, `find` has the `-exec` option, which executes a command on each file it finds. I'll use `find` to `cat` all files under the ".doormat" directory for me:

![Deepdir - find exec](/images/2016/12/deepdir_find_exec.png)

Done! The key for the next door is:

**open_sesame**

### Sant's Office Terminal - Chess?
Inside Santa's office I found the next terminal. Instead of a shell prompt, I got a greeting. The first thing I tried was a "?" and got another response:

![Santa's Workshop - Greetings](/images/2016/12/santasworkshop_greetings.png)

Turning to Google, I realized that "GREETINGS PROFESSOR FALKEN" is a reference to [Wargames](http://www.imdb.com/title/tt0086567/). I found the exact clips on Youtube where the main character is interacting with the computer:

https://www.youtube.com/watch?v=D-9l5jSDL50
https://www.youtube.com/watch?v=-1F7vaNP9w0

If I freeze framed, I could see exactly what was typed on screen. The terminal was very picky on capitalization and punctuation, but I just kept typing out what Matthew Broderick did and eventually started the game:

```
GREETINGS PROFESSOR FALKEN.

Hello.

HOW ARE YOU FEELING TODAY?

I'm fine. How are you?

EXCELLENT, IT'S BEEN A LONG TIME. CAN YOU EXPLAIN THE REMOVAL OF YOUR USER ACCOUNT ON 
6/23/73?

People sometimes make mistakes.

YES THEY DO. SHALL WE PLAY A GAME?

Love to. How about Global Thermonuclear War?

WOULDN'T YOU PREFER A GOOD GAME OF CHESS?

Later. Let's play Globalthermonuclear War.

FINE
```

Once the game started, I copied the moves from the movie. I chose Soviet Union, and when it came time to choose targets, I chose the first city chosen in the movie: Las Vegas.

![Santa's Office - Key](/images/2016/12/santasworkshop_key.png)

The key for the door in Santa's Workshop (which is behind the left bookshelf, btw) was:

**LOOK AT THE PRETTY LIGHTS**

### Train Terminal - OUTATIME
The door behind Santa's office did not have a console, so I saved that one for later and instead went to the last terminal by the train.

I was placed in a "Train Management Console":

![Train - Management Console](/images/2016/12/train_mgmt_console.png)

The first thing I did was read the HELP:

![Train - Help LESS](/images/2016/12/train_less.png)

What immediately jumped out at me was the filename highlighted in the bottom left. I was inside the `less` command. The clue on the 6th line confirms it ('less' is unnecessarily capitalized).

I've used `less` enough to know there are several different commands you can use from within it. You can read more about `less` [here](http://www.tutorialspoint.com/unix_commands/less.htm).

From within `less`, I can open and view other files with the `:e` command. Tab complete even works when opening files, so I tab-completed to see what else was inside the `/home/conductor` folder. There were two other files: "ActivateTrain" and "Train_Console":

![Train - Less Open](/images/2016/12/train_less_open.png)

Opening that file revealed the password necessary to start the train:

![Train - Password](/images/2016/12/train_less_password.png)

It looked like a hash, but I verified in the code that when the comparison is made it was not hashing my input, but rather just doing a string comparison. So that was the actual password:

**24fb3e89ce2aa0ea422c3d511d40dd84**

With that password, I could dis-engage the brakes and start the train to go back in time to 1978!

![Train - Flux Capacitor](/images/2016/12/train_flux_capacitor.png)

Once in 1978, I made my way back to Santa's workshop and found him locked in the reindeer pen:

![Found Santa](/images/2016/12/train_found_santa-1.png)

He didn't remember how he had gotten there, so I still needed to find out how kidnapped him...

> 5) What is the password for the "cranpi" account on the Cranberry Pi system?

**yummycookies**

>6) How did you open each terminal door and where had the villain imprisoned Santa?

**See Above Sections :)**

## Part 4: My Gosh... It's Full of Holes
Now that I had found Santa in-game, I had to compromise targets identified in the APK to recover the other audio files. The [online write up](https://holidayhackchallenge.com/2016/) outlines 6 targets:

* The Mobile Analytics Server (via credentialed login access)
* The Dungeon Game
* The Debug Server
* The Banner Ad Server
* The Uncaught Exception Handler Server
* The Mobile Analytics Server (post authentication)

My first step was to identify where these servers are. Going back to my decompiled APK, I just did a recursive grep for "http://" and "https://" to see what URLs were hardcoded in the application. I found several URLs defined in `res/values/strings.xml`:

![Android strings.xml](/images/2016/12/android_strings_urls-1.png)

I verified the associated IP addresses of these URLs with Tom Hessman (as instructed) and had my list of servers to target:

* **Analytics** -https://analytics.northpolewonderland.com
* **Dungeon** - http://dungeon.northpolewonderland.com
* **Debug** - http://dev.northpolewonderland.com/index.php
* **Ads** - http://ads.northpolewonderland.com
* **Exception Handler** - http://ex.northpolewonderland.com/exception.php



### Analytics
*Note: I'm going to write up my solutions in the order that I found them. I focused on the analytics server first and didn't move on until I had both mp3 files from it*

#### Analytics - Part 1
Visiting the analytics server at https://analytics.northpolewonderland.com/, I was presented with a login page:

![Analytics - Login Page](/images/2016/12/analytics_login.png)

Fortunately, I had already recovered some hardcoded credentials from the APK file way back in Part 2: 

**guest / busyreindeer78**

Using these creds, I was able to login to the server and see a link to download an mp3 file:

![Analytics - mp3 Download](/images/2016/12/analytics_mp3_1.png)

https://analytics.northpolewonderland.com/getaudio.php?id=20c216bc-b8b1-11e6-89e1-42010af00008

Easy enough! Two mp3's down (I had one from the APK), five to go.

#### Analytics - Part 2
At this point, I decided to keep focusing on the Analytics server. Seeing a PHP application with "Query" functionality just screamed SQLi to me so I explored that path.

I used BurpSuite to capture requests while I explored the site functionality. I was able to run custom queries by making POST requests like this:

![Analytics - Query POST](/images/2016/12/analytics_post.png)

Since I wasn't really concerned about being stealthy or careful, I YOLO'd it and sent that entire request to SQLmap, certain I was about to dump the database through SQLi:

```bash
$ sqlmap -r query.req --force-ssl
```

Imagine my shock when sqlmap came back with no vulnerable parameters! Well there was also a "view" functionality that takes a GET param. Surely that was the one? Nope, same results from sqlmap.

At this point I started to think maybe the SQLi (and I was still convinced it was SQLi) wasn't that easy. I was still only the "guest" user, maybe I need to focus on privilege escalation.

I decided I needed to step back and do some more reconnaissance on the server. Maybe there was another service listening or some other pages on the site that I couldn't reach through the home page. I was hoping for something like "/admin" or "/manage", or an exposed API.

I ran an nmap scan with the `-A` option to enumerate everything I could:

![Analytics - Nmap](/images/2016/12/analytics_nmap.png)

An exposed Git directory! I've seen this several times on external pentests, and it almost always leads to something good :)

I visited the site in my browser and was pleasantly surprised that not only was the .git folder present and unprotected, but nginx was indexing it for me:

![Analytics -  Git Dir](/images/2016/12/analytics_git.png)

Since Dir Indexing is enabled, it was easy to grab an entire copy of the .git directory. There's an excellent writeup from Netspi on [Dumping Git Data from Misconfigured Web Servers](https://blog.netspi.com/dumping-git-data-from-misconfigured-web-servers/).

Had directory indexing not been enabled, it's still possible to get some really good info from the directory. There's a great tool called [DVCS-Ripper](https://github.com/kost/dvcs-ripper) that automates the process.

To pull down a local copy of the entire git repo, I used wget:

```bash
$ wget -r https://analytics.northpolewonderland.com/.git/
```

Then once, downloaded. I was able to `cd` into the directory and do a `git reset --hard` to get the latest source code for the web app:

![Analytics - Git Checkout](/images/2016/12/analytics_git_checkout.png)

Now that I had the source code, I started reading through it to better understand the app. Although there were a lot of SQL queries, the app seemed to always be using the `mysqli_real_escape_string` function to prevent straightforward SQLi.

One page in the source code that I hadn't seen yet was `edit.php`. Trying to view it I got "Access denied!" and looking through the source, I see that only the "administrator" user would be able to access it.

![Analytics - edit access denied](/images/2016/12/analytics_edit_denied.png)

At this point, I knew I had to elevate myself to the "administrator" user. Authorization was being handled through the "AUTH" cookie, and the username was encrypted in the value. The relevant PHP code was in the functions "get_username" in `db.php`:

```php
function get_username() {
    if(!isset($_COOKIE['AUTH'])) {
      return;
    }

    $auth = json_decode(decrypt(pack("H*",$_COOKIE['AUTH'])), true);

    return $auth['username'];
  }
```

and `crypto.php`:

```php
<?php
  define('KEY', "\x61\x17\xa4\x95\xbf\x3d\xd7\xcd\x2e\x0d\x8b\xcb\x9f\x79\xe1\xdc");

  function encrypt($data) {
    return mcrypt_encrypt(MCRYPT_ARCFOUR, KEY, $data, 'stream');
  }

  function decrypt($data) {
    return mcrypt_decrypt(MCRYPT_ARCFOUR, KEY, $data, 'stream');
  }
?>
```

A hardcoded encryption key! This key is used to encrypt and decrypt the AUTH cookie. I now had everything I needed to forge a cookie (hopefully). First, I decrypted my AUTH cookie for the "guest" account by importing `crypto.php` in an interactive PHP session and following along with the above `get_user` code:

![Analytics - Cookie Decrypt](/images/2016/12/analytics_cookie_decrypt.png)

*Note: I had to `apt-get install php-mcrypt` in Kali*

Looked pretty straightforward. The cookie was just a JSON string. I changed the "username" to "administrator" and re-encrypted:

![Analytics - encrypt cookie](/images/2016/12/analytics_cookie_encrypt.png)

Armed with a new cookie, I used the [Cookies Manager+ Addon](https://addons.mozilla.org/en-US/firefox/addon/cookies-manager-plus/) to change my cookie:

![Analytics - change cookie](/images/2016/12/analytics_cookie_set.png)

I refreshed, and voila - a new page to attack!

![Analytics - edit page](/images/2016/12/analytics_edit_page.png)

This new page let me edit an existing query's name and description by entering its ID. I got a query ID by running a launch query and checking "Save this query". Viewing the results showed me the details, including the ID of the query I just ran:

![Analytics - Query Details](/images/2016/12/analytics_query_details.png)

I edited this query to change the name and description to "foobar". The request looked like this:

![Analytics - Edit Request](/images/2016/12/analytics_edit_request.png)

What's nice is the page actually shows you the SQL that was executed to modify the DB:

![Analytics - Edit SQL](/images/2016/12/analytics_sql_printed.png)

The SQL statement showed that the application is building an UPDATE query from my GET parameters.

Naturally, since I'm lazy, I YOLO'd it again and fed that request into sqlmap. Unfortunately I didn't get any results. I was seriously starting to doubt this was a straightforward SQL injection, so I decided I needed to take a closer look at the source code, specifically for this new `edit.php` page. Every SQL query was correctly using `mysqli_real_escape_string`, so SQLi probably wasn't possible. However, as I worked my way through the code I started piecing together what it was doing and noticed a vulnerability:

![Analytics - edit.php](/images/2016/12/analytics_edit_php_marked.png)

(I marked up the interesting bits and I'll explain below)

1. It takes the `id` get parameter and pulls the first matching query from the database. If the row doesn't exist, it aborts.
2. It iterates over each value (column name) in the row and checks to see if there is a GET parameter with that name (`if(isset($_GET[$name]))`). It prints "Checking for (columnName)" and "Yup!" if there's a matching GET param.
3. If there is a GET parameter that matches the column name, it adds the string "(columnName) = (getParamValue)" to a `$set` variable.
4. It builds a SQL UPDATE statement by joining all the strings in `$set` with a comma, and
5. Adding a WHERE clause with the supplied `id` param
6. It prints the statement and executes the query

The interesting part is that even though the "edit" page only lets you enter "name" and "description", the code will actually iterate over all GET params and add them to the SQL statement if they match column names. 

Since the SQL Schema is included in the source code (`sprusage.sql`), I knew that the reports table had a column named "query". It's also verified on the output page of edit ("Checking for *query*...").  I assumed this "query" value was actual SQL that was executed, and verified it by looking at `view.php` where it executes the SQL statement directly from the "query" value:

```php
format_sql(query($db, $row['query']));
```

Using Burp repeater, I added a GET parameter for "query". I decided to try a simple MySQL query to get the hostname of the server: `SELECT @@hostname;` Since "query" is both a column name and a GET parameter now, the app added it to the UPDATE statement.Looking at the response, it looks like the SQL statement worked and I changed the query for the report with ID `ae8ad368-46ff-4bad-88bd-fba7d055cafd`:

![Analytics - select hostname](/images/2016/12/analytics_select_hostname.png)

Now all I had to do was the view the results of that report by visiting https://analytics.northpolewonderland.com/view.php?id=ae8ad368-46ff-4bad-88bd-fba7d055cafd

![Analytics - Hostname](/images/2016/12/analytics_hostname_result.png)

Woohoo! Arbitrary SQL execution! At this point my mind started racing and wondering if I could get local code execution by writing PHP or maybe reading sensitive system files, but then I remembered the whole point of this challenge was just to get an mp3 file...

Looking back at the SQL schema in the Git repo, I knew mp3 files were stored in a table called audio:

![Analytics - audio sql](/images/2016/12/analytics_audio_mp3.png)

So I used Burp Repeater to update the query again for this report to be `SELECT * from audio;` and this was the output:

![Analytics - audio output](/images/2016/12/analytics_audio_output.png)

Excellent, now I had the ID and filename for the audio file. I tried using the same `getaudio.php` link I used for the first mp3 file and changing the ID, but got this:

![Analytics - mp3 no access](/images/2016/12/analytics_mp3_no_access.png)

Looking at the source code for `getaudio.php` I realized the problem. I could only access the page if I was the "guest" user, and I couldn't download the mp3 file I needed because the username defined in that row was "administrator". 

I got frustrated because I knew where the audio file was but couldn't access it. Then I realized that in my report output, there was a column for "mp3" but no content was displayed. I assumed this was because the mp3 column was type `MEDIUMBLOB` and the `format_sql` command in `view.php` didn't know how to output it. I looked around online and realized that MySQL has a built in `HEX()` function. So once again I used Repeater to change the report query to: `SELECT HEX(mp3) FROM audio WHERE username='administrator';`. That should pull just the mp3 file I care about and hex encode it for me. Viewing the report again:

![Analytics - hex mp3 file](/images/2016/12/analytics_hex_mp3.png)

There we go! The hex encoded mp3 file! I viewed-source and copied the entire hex string to a file. Then simply used `xxd` and got the raw mp3 file back!

```bash
$ xxd -r -p mp3_hex > discombobulatedaudio7.mp3
$ file discombobulatedaudio7.mp3
discombobulatedaudio7.mp3: Audio file with ID3 version 2.3.0, contains: MPEG ADTS, layer III, v1, 128 kbps, 44.1 kHz, JntStereo
```

Done with the analytics server!

### Ads Server
Next I looked at the ads server located at http://ads.northpolewonderland.com/

Running it through Burp, I saw a request for a static JS file: 

http://ads.northpolewonderland.com/fedc8e9f69dab9d81a4f227d6ec76567fcb56231.js?meteor\_js\_resource=true

I had not heard of the [Meteor Framework](https://www.meteor.com/) before, but one of the in-game elves gave some really good hints about it. Specifically, this blogpost about abusing misconfigurations:

https://pen-testing.sans.org/blog/2016/12/06/mining-meteor

Like most client-side app frameworks, if the developer isn't careful, too much sensitive information can be sent from the server and stored client-side. The author of the blog released an awesome Tampermonkey script called [MeteorMiner](https://github.com/nidem/MeteorMiner) to help in parsing and viewing data from Meteor subscriptions. 

I installed the extension and re-visited the page. I now had a pretty graphical representation of collections, routes, subscriptions, etc from the Meteor app:

![Ads - Meteor Miner](/images/2016/12/ads_meteor_miner.png)

Using my browser console, I could view the contents of the collections, for example "Quotes":

![Ads - Quotes Collection](/images/2016/12/ads_quotes_collection.png)

These appeared to be the quotes that were displayed on the home page. I didn't see anything interesting in either the "HomeQuotes" or "Satisfaction" collections, so I started to look at the other pages. The nice thing about MeteorMiner is that it lists the "Routes", or pages that I could visit. I clicked through to each one and MeteorMiner kept updating with the collections/subscriptions on that page.

On the page "/admin/quotes", I noticed a new Subscription: "adminQuotes". Despite not being logged in and the page not displaying, it looked like Meteor was still fetching some data. I looked at the "HomeQuotes" collection again using my browser console:

![Ads - Admin Quotes](/images/2016/12/ads_admin_quotes.png)

It contained the path to the next mp3 file! http://ads.northpolewonderland.com/ofdAR4UYRaeNxMg/discombobulatedaudio5.mp3

The MeteorMiner extension and great SANS blogpost made this challenge pretty easy. I hadn't known about Meteor before this, but now I can't wait to come across it on an assessment and test this out "for real" :)


###  Debug Server
Next I targeted the "debug" server located at:
http://dev.northpolewonderland.com/

I did some initial recon with nmap and nse scripts, but found nothing interesting. There was an "index.php" page but it was blank. I used Burp and tried various HTTP verbs with various payloads, but never got anything other than a blank 200 Response from the server. 

I knew the Android application must communicate with the server (I got the URL from the APK after all), so I decided to do some dynamic analysis on the application. I was at my parents house and didn't have an Android device with me, so I downloaded a personal edition of [Genymotion](https://www.genymotion.com/) to quickly spin up virtual Android devices. I used a virtual device with API level 18 (JellyBean) and installed SantaGram using adb:

```bash
$ adb devices
List of devices attached
192.168.56.101:5555	device

$ adb install SantaGram_4.2.apk
[100%] /data/local/tmp/SantaGram_4.2.apk
	pkg: /data/local/tmp/SantaGram_4.2.apk
Success
```

I wanted to intercept all the HTTP/S traffic from SantaGram and hopefully see a request go to the Debug server. If you're reading this and haven't used Burp with an Android device before, I strongly recommend [this blogpost](https://blog.bugcrowd.com/mobile-testing-setting-up-your-android-device-part-1) from Bugcrowd. 

Once I had SantaGram installed I opened it up and saw requests start coming through Burp. Phew! SantaGram doesn't do cert-pinning :)

I was presented with a login page, but there was an option to create an account, so I registered a test account:

![Debug - Register](/images/2016/12/debug_register-1.png)

Once I had an account, I clicked around and on everything I possibly could. I updated my profile, I searched for people, and just generally tried to do absolutely everything the app let me. What I wanted was to generate as much traffic as I could in Burp to (hopefully) catch a debug request.

Sadly, in all my clicking, I never saw a single request to dev.northpolewonderland.com. That didn't make sense to me - the URL was in the APK, when did it talk to the server?

I looked again at `strings.xml` where the server was defined and saw the reason why:

```xml
<string name="debug_data_collection_url">http://dev.northpolewonderland.com/index.php</string>
    <string name="debug_data_enabled">false</string>
```

Debug was set to "false" in the application. I bet if I switched it to "true" I would start seeing requests! To change it, I would need to modify the value in the XML value, re-compile the application, and re-install it on my device.

I've modified Android applications before (usually to disable cert-pinning) and always referred to this [great blog series](https://pen-testing.sans.org/blog/pen-testing/2015/06/30/modifying-android-apps-a-sec575-hands-on-exercise-part-1) from SANS on modifying Android apps.

First, I used `apktool d` to decompile the application, then modified the `strings.xml` file to change the value of `<string name="debug_data_enabled">false</string>` to "true". Then I used `apktool b` to re-build the application:

![Debug - Modify app](/images/2016/12/debug_modify_app.png)

The rebuilt APK is in the `dist/` directory. But it can't be installed until it's signed. To sign it, I generated a self-signed cert using Java's `keytool` and then signed the APK using `jarsigner`:

![Debug - Sign App](/images/2016/12/debug_sign_app.png)

Then I uninstalled the "stock" SantaGram app from my device, and installed the modified one using adb:

```bash
$ adb install SantaGram_4.2/dist/SantaGram_4.2.apk
```

Now I used the app as before (clicking on everything and capturing traffic in Burp). This time, however, I finally saw a Debug request!

![Debug - POST Request](/images/2016/12/debug_post_request.png)

Excellent! Now I had a request I could replay to the server that actually gave me a response! I sent the request to Repeater and started tampering with some parameters. I wanted to see what was reflected back to me and if I could cause any verbose errors:

![Debug - POST Response](/images/2016/12/debug_post_response.png)

I never saw any errors, but noticed a few interesting things. The entire "request" was echoed back to me in the response, and there was reference to a "filename". Pointing my browser to the filename, I saw the request echoed back to me in a txt file:

![Debug - Text file](/images/2016/12/debug_file_txt.png)

Alright, so I can write values to a text file on the server - so what? Looking at the response in Burp, the content-type is correctly set as `text/plain`, so injecting JavaScript or PHP probably won't work....

I went back to Repeater and started trying to inject different special chars hoping I could cause something interesting to happen. I was really hoping for a PHP error that leaked info or maybe a SQL exception. That's when I noticed something I missed the first time. In the response, when it echoes back the "request", it has a parameter named "verbose" that I didn't send. Where did that come from?

On a hunch, I resent the same request but specified `"verbose":true`:

![Debug - Verbose True](/images/2016/12/debug_request_verbose-1.png)

Nice! Manually setting "verbose" to true spit out a bunch of information, including files on the server :)

Now I had the next mp3:

http://dev.northpolewonderland.com/debug-20161224235959-0.mp3

###  Exception Handler Server
Next up I decided to target the exception handler server:

http://ex.northpolewonderland.com/exception.php

Looking through my Burp state, I didn't see any requests to that server from my Android device, so I was hoping there was enough to go on just looking at the server (unlike Debug where I needed to capture a valid request)

First I noticed when visiting the site that only POST requests were accepted:

![Exception - POST only](/images/2016/12/exception_post_only.png)

Burp it is. I sent the request to Repeater and right clicked to select "Change Request Method". Now I got an error:

![Exception - Must be JSON](/images/2016/12/exception_must_be_json.png)

That's useful. So it's expecting to receive some JSON data via POST. I set the content-type to "application/json", sent some dummy JSON and got another error:

![Exception - Operation Key needed](/images/2016/12/exception_operation_key.png)

Very useful! These verbose error messages are telling me exactly what params I need to send. I added `"operation":"WriteCrashDump"` and got an error that "data" was missing. So I added `"data":"foobar"` and finally got a successful response:

![Exception - Write Success](/images/2016/12/exception_write_success.png)

It looked like I had written a PHP file to the "docs" folder? I confirmed it with my browser:

![Exception - Debug File](/images/2016/12/exception_debug_file.png)

Yep! Looked like my "data" parameter. Knowing I could write to a PHP file in the web directory, the next thing I tried was sending `"data":"<?php echo 1;?>"` to see if I could execute arbitrary PHP code:

![Exception - PHP Response](/images/2016/12/exception_php_response.png)

No, I can see that it's just returning the content of "data" wrapped up in quotes as HTML. I tried sending some special characters to see if I could "break out" of the string, but no luck.

Then I remembered there was another "operation" available to me when making POST requests: `ReadCrashDump`. I changed the operation to "ReadCrashDump" and the server responded that another key was needed: "crashdump". I set that one to "foobar" as well and got an interesting error:

![Exception - Crashdump Needed](/images/2016/12/exception_crashdump_key.png)

It is set! Why doesn't it recognize it? Unless it's looking for it in the wrong place... It wanted a "data" parameter too...maybe "crashdump" needs to *inside* that one:

![Exception - Crashdump 500](/images/2016/12/exception_crashdump_500.png)

Well now it 500's. But my hunch was confirmed - it didn't complain about the "crashdump" key missing. Obviously, "foobar" isn't a valid crashdump to read. Thankfully I had written one earlier ("crashdump-d6I79O.php") so I tried to read it:

![Exception - duplicate extension](/images/2016/12/exception_duplicate_extension.png)

Oh cool! Now it complained about a duplicate PHP extension. That told me there was server-side code trying to add ".php" to the end of the value of "crashdump" and then trying to read it. Removing the ".php" from my crashdump parameter spits out the correct content. This had local file inclusion written all over it! First, I wanted to see if it honored path traversal using "..". Since I was reading crashdump files in the "docs" directory, could I read PHP files in the parent directory (the web root)?

![Exception - path traversal](/images/2016/12/exception_dir_traversal-1.png)

Perfect, I could read files from the parent directory. If I removed the extension from "exception.php", I would get a 500 error though and nothing would be displayed.

Talking to the elves in game, I got a big hint and a great resource about PHP LFI vulnerabilities in the form of another excellent [SANS blogpost](https://pen-testing.sans.org/blog/2016/12/07/getting-moar-value-out-of-php-local-file-include-vulnerabilities). In the post, the author talks about using the `php://filter/base64-encode/resource=` stream filter to display contents as base64. As the post discussed, it can allow PHP source code to be exfiltrated instead of parsed and ran when doing LFI on PHP files. It sounded like exactly what I needed. I wanted to read the source code of `exception.php` so I passed the php filter as the "crashdump" parameter to the "ReadCrashDump" operation:

![Exception - Base64 Filter](/images/2016/12/exception_base64_filter.png)

w00t! It spit out a bunch of base64 encoded data. Highlighting it and sending it to Burp Decoder, I could see it was the source code of `exception.php`:

![Exception - Source Code](/images/2016/12/exception_source_code.png)

And right there in a comment is the location of the next mp3 file :)

http://ex.northpolewonderland.com/discombobulated-audio-6-XyzE3N9YqKNH.mp3

Six mp3s down, one more to go!

### Dungeon
*Note: This challenge by far took me the longest, but it turns out it took me so long because SANS was having some server issues...once the server was working I realized I had the solution pretty early on. Oh well :/*

The challenge I saved for last was Dungeon. One of the elves gave you an "older" version of the Dungeon game, located here:

www.northpolewonderland.com/dungeon.zip

The URL from the APK pointed you to directions for the game:

http://dungeon.northpolewonderland.com/

Dungeon was an ELF64 binary and looked like a text based adventure game:

![Dungeon - Start](/images/2016/12/dungeon_start.png)

There was also another file in the Zip called `dtextc.dat`. I wasn't sure what kind of file it was or what data it contained yet, but if I removed or renamed it the Dungeon game wouldn't work, so I knew it contained important information that was necessary for the game.

Just like Wumpus, the last thing I wanted to do was play a text based game, so I decided to cheat. Since I had a local copy of the binary this time, I threw it into [Hopper](https://www.hopperapp.com/) to take a look at the disassembly. I figured the link to the next mp3 had to be somewhere in the binary. I looked the strings that Hopper found, but found nothing related to an mp3. One string jumped out as me (as it would to any good pentester): "You are not an authorized user"

![Dungeon - Hopper Strings](/images/2016/12/dungeon_hopper_strings.png)

The string above it looked like it could be a hardcoded password also. In Hopper, pressing 'x' on a string let's you see where it is referenced and jump to it. I followed the reference and it was in a function called `_gdt`:

![Dungeon - GDT Function](/images/2016/12/dungeon_gdt_function.png)

I felt like this was an important function, so I started tracing it (by pressing 'x') to see when/if it was called from within the game. It's called only once: inside the `game_` function at offset `4049a7` (highlighted in blue at the bottom):

![Dungeon - GDT Call](/images/2017/01/dungeon_gdt_call.png)

Working my way up the instructions, I can see that it's only called if a `strcmp` comes back true (at `40499e`). Since this is the main function of the game, and it's calling `rdline_` earlier, I assumed this was checking my input on the main line. It's not easy to tell just by looking at the assembly what it's comparing with `strcmp`, so I used a debugger (GDB). I use the awesome [GDB Peda](https://github.com/longld/peda) framework to show me much more relevant information such as register contents, stack contents, and guessed arguments for function calls.

I set a breakpoint on the `strcmp` call at `40499e` and ran the game. When I entered "?" and hit enter, my breakpoint hit and I could see what it was comparing my input ("?") against:

![Dungeon - GDB Compare](/images/2017/01/dungeon_gdb_strcmp.png)

Duh, easy enough! It's checking to see if I entered "GDT". So that's how you call this mysterious "GDT" function...by typing it's dang name! (I kinda felt stupid for not guessing that...)

I ran the game normally this time and entered "GDT". Then I tried "?" and "HELP" and got a list of commands I can use inside GDT (I also saw these in the strings in the binary):

![Dungeon - GDT Help](/images/2017/01/dungeon_gdt_help.png)

Looking at what I could do, the command that seemed most valuable to me was "Display text". I was still convinced the URL of the mp3 file was buried/encrypted/encoded somewhere in this game. Typing "DT" it prompted me for an "Entry". I took a super scientific guess and entered "1", and then "1000":

![Dungeon - GDT DT 1](/images/2017/01/dungeon_gdt_dt_1.png)

Now these didn't look too interesting at first to me, but then I realized I had never seen these strings before. They were not in the binary - they must be being read/decrypted from the .dat file!

Alright, I thought, the path to the mp3 file has to be somewhere in there and I though I could just use "DT" to print it out. I just needed to know the Entry number. Thinking it might be the very last one, I tried to find the maximum entry number I could enter. 

First I tried 2000 and got a Segmentation Fault. Then I tried 1500 and got a Segmentation Fault. 1250? No SegFault, but nothing displayed. 1125? Nothing. 1063? Nothing. 1032? Nothing. 1016? Got a string. Then I just bounced between 1016 and 1032 until I found the "last" string, 1027:

![Dungeon - GDT 1027](/images/2017/01/dungeon_gdt_1027.png)

Alright! That looks like the string that's displayed when someone beats the game. I worked my way backwards a few entries, but still didn't see any reference to an mp3. I was still convinced it had to be in there somewhere, so I decided to spit out every text entry between 1 and 1027.

Now instead of typing "DT" 1027 times, I wanted to script it. I thought at first about using GDB, then realized there was a much easier way using a tool that's saved me many times: [expect](http://www.tcl.tk/man/expect5.31/expect.1.html)

`expect` lets you interact with programs on the command line and send/receive input. Since I knew exactly what to type to get the string displayed ("GDT\nDT\n1\n") I could use `expect` to send the keystrokes for me and display the output to the STDOUT. I had to google a little to figure out how to [do a loop](http://www.thegeekstuff.com/2011/01/expect-expressions-loops-conditions/) in `expect`, but ultimately got a script together that would run "DT" 1026 times from the command line:

```bash
#!/usr/bin/expect

spawn "./dungeon"

expect ">"
send "GDT\n"
expect "GDT>"

for {set i 1} {$i < 1030} {incr i 1} {
send "DT\n"
expect "Entry:"
send "$i\n"
expect "GDT>"
}
```

The script basically runs the dungeon binary, waits for the prompt (">"), enters "GDT\n", waits for that prompt, then loops through entering 1-1030 for "DT". Everything is spit out to STDOUT so I used `tee` to capture it to a file:

![Dungeon - Expect GDT](/images/2017/01/dungeon_expect_gdt_output.png)

Using `vim` I stripped out the first few lines, and the "GDT>DT" and "Entry: " lines to just leave me with every string from the game.

I browsed through the file and was quite dismayed when there was no reference to an mp3 file :(

Instead, the only real interesting string that jumped out at me was:

```
The elf, satisified with the trade says -
Try the online version for the true prize
```

Reading this I thought there might be an online version of this Dungeon game I needed to "play". I never really explored the server `dungeon.northpolewonderland.com`, so I kicked off an Nmap scan on the server:

![Dungeon - Nmap Scan](/images/2017/01/dungeon_nmap_scan.png)

*Note: this is where I got stupidly stuck when I first tried it. Nmap didn't return anything other than port 80 open and there was nothing in the webserver except the instructions. I went back into the online minigame thinking the game existed there, but couldn't find it. So I turned back to the local copy and kept trying to reverse it looking for something else. Turns out there is an online version, SANS was just having network troubles and it was down when I ran nmap. I'm gonna skip ahead and pretend the service was up from the start for me*

Port 11111 seemed odd, so I tried connecting with `netcat`:

![Dungeon - Online Version](/images/2017/01/dungeon_online-1.png)

There it is! An online version of the game. I checked and GDT worked and I could spit out strings. I wanted to dump everything from the online version and diff to my local version, so I wrote a Python script to repeatedly run "GDT>DT" (just like my expect script) but over a network socket:

```python
#!/usr/bin/env python
import socket

HOST = "dungeon.northpolewonderland.com"
PORT = 11111

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((HOST, PORT))

print s.recv(1024)
s.send('GDT\n')
print s.recv(1024)
for i in range(1,1028):
    s.send('DT\n')
    print s.recv(1024)
    s.send('{}\n'.format(i))
    print s.recv(1024)
s.close()
```

I ran this script and tee'd the output to another file as well.

```bash
$ python online_gdt.py |tee online_gtd_output.txt
```

Now that I had all the strings from both the local version AND the online version, I used a visual editor to diff them and look at the differences. They were almost identical, but one change jumped out at me:

![Dungeon - GDT Diff](/images/2017/01/dungeon_gdt_diff.png)

So instead of going online for the prize, I just "send email to peppermint@northpolewonderland.com".

I sent a blank email to that address (no subject, no body) and got an auto-reply!

![Dungeon - Email Reply](/images/2017/01/dungeon_gdt_email.png)

YES! I now had the final mp3 and had completed all the challenge servers :D

## Summary of Part 4
To summarize, I now had 7 audio files from the different targets:

- **SantaGram APK**. The mp3 file <u>*discombobulatedaudio1.mp3*</u> was inside the APK at `res/raw/discombobulatedaudio1.mp3`

- **Analytics Server (via credentialed access)**. I used credentials hardcoded in the APK at `com/northpolewonderland/santagram/SplashScreen.class` to login to https://analytics.northpolewonderland.com/. The mp3, <u>*discombobulatedaudio2.mp3*</u> was linked on the home page

- **The Dungeon Game**. Using a special function, `GDT`, I was able to display decoded/decrypted text from the associated `.dat` file. Strings pointed me to an online version of the game at `dungeon.northpolewonderland.com:11111`. Using a Python script I dumped the strings from the online version using `GDT` and saw instructions to email peppermint@northpolewonderland.com. The auto-reply gave me <u>*discombobulatedaudio3.mp3*</u> as an attachment.

- **The Debug Server**. To generate debug requests, I modified `res/values/strings.xml` in the APK to change `debug_data_enabled` to `true`, then recompiled and self-signed the app. I ran it in Genymotion and captured debug requests to http://dev.northpolewonderland.com/index.php through Burp. I observed a parameter in the response called `verbose` and when I replayed a request specifying that parameter and setting it to `true`, the response listed files on the server, including <u>*debug-20161224235959-0.mp3*</u>

- **The Banner Ad Server**. The ads server at http://ads.northpolewonderland.com/ used the Meteor Framework to run a client side application. I used [MeteorMiner](https://github.com/nidem/MeteorMiner) to view routes, and on `/admin/quotes`, found a link to the mp3 file in the "HomeQuotes" collection which led me to <u>*discombobulatedaudio5.mp3*</u>

- **The Uncaught Exception Handler Server**. Using verbose errors which told me what keys/params were missing, I was able to build valid POST requests to http://ex.northpolewonderland.com/exception.php. The `ReadCrashDump` operation was vulnerable to Local File Inclusion when reading crashdump PHP files in the "docs" folder. I used the php filter technique [here](https://pen-testing.sans.org/blog/2016/12/07/getting-moar-value-out-of-php-local-file-include-vulnerabilities) to read the source code for `exception.php`. In a comment in the source was a link to <u>*discombobulated-audio-6-XyzE3N9YqKNH.mp3*</u>

- **Mobile Analytics Server (post auth)**. An unprotected and indexed Git directory, https://analytics.northpolewonderland.com/.git/, let me grab source code for the application. Using the hardcoded encryption key, I forged a cookie for the "administrator" user and gained access to the `edit.php` page. Looking at the source code for that page, I saw it was possible to modify the query for an existing report and execute arbitrary SQL statements. From the `.sql` file, I knew the mp3 existed as a binary blob in the database, so I executed `SELECT HEX(mp3) FROM audio WHERE username='administrator';` in a report to display and download a hex encoded version of <u>*discombobulatedaudio7.mp3*</u>


## Part 5: Discombobulated Audio
I now had all seven mp3 fragments! I renamed the two files that had different filenames to match the "discomboluatedaudioX.mp3" format:

![Audio - All Files](/images/2017/01/audio_all_files.png)

To combine them all, I simply used `cat` and redirected output to a new file:

```bash
$ cat discombobulatedaudio* > combined.mp3
```

Listening to the combined audio file, it sounded like it had really been slown down. So I loaded it up in [Audacity](http://www.audacityteam.org/download/) and looked at the waveform:

![Audio - Audacity](/images/2017/01/audio_audacity.png)

Within Audacity, I went to "Effect -> Change Speed" and sped it up by 400%. Unfortunately when I played it it still sounded awful. I realized that by changing the speed I was also changing the pitch so what I thought might be "voices" were just high pitched shrieks. 

I Googled how to change speed without changing pitch and learned that Audacity has the feature built in called "Filter->Change Tempo". I undo'd the change in speed and instead did a 400% increase on "Change Tempo"

![Audio - Change Tempo](/images/2017/01/audio_change_tempo.png)

This time when playing it, I understood voices! It was still a little slow and a bit distorted, but I could clearly make out some words:

"?? Christmas...Santa Claus...or as I've ?? known him...??"

There seemed like there was some background music too. It sounded like a quote from a movie, so  I googled the words I could pick out:

![Audio - Google Results](/images/2017/01/audio_google_quote.png)

The top result was a quote from an episode of Dr. Who called "A Christmas Carol":

> The Doctor: Father Christmas, Santa Claus. Or, as I've always known him, Jeff.

I've never watched Dr. Who, so I was hoping at this point I didn't need to know anything else about Dr. Who and that this was all I needed to get through the final door.

I went back in the mini-game and went to the final locked door in the corridor behind Santa's office. It prompted me for a password. I started taking some guesses:

- "Dr. Who"
- "The Doctor"
- "Jeff"
- "A Christmas Carol"
- "Tardis" (that's a Dr. Who thing right?)

Nothing was working and I was starting to think I missed something. I was even considering watching the whole episode of Dr. Who to see if there was another reference or clue. Finally I just tried the whole quote, with punctuation:

- "Father Christmas, Santa Claus. Or, as I've always known him, Jeff."

It worked! I got through the door, went up the ladder and met Santa's kidnapper face to face:

![Audio - Dr. Who](/images/2017/01/audio_dr_who.png)

Found him! It was Dr. Who who kidnapped Santa Clause. Talking to him he reveals why:

![Audio - Dr. Who Explanation 1](/images/2017/01/audio_dr_who_text.png)

![Audio - Dr. Who Explanation 2](/images/2017/01/audio_dr_who_text2.png)

He just wanted to create a world where the "Star Wars Holiday Special" never existed and I foiled it. Sorry :/
 >9) Who is the villain behind the nefarious plot.

Dr. Who

>10) Why had the villain abducted Santa?

He wanted to use Santa's magic to stop the "Star Wars Holiday Special" from being created. Santa didn't agree with him so he kidnapped him.

## Summary
So after countless hours, I had finally completed all the challenges and found Santa's kidnapper.

I had an absolute blast playing the hack challenge. Ed Skoudis and the entire SANS team really went above and beyond to make an entertaining, but still realistic CTF style challenge. I especially liked that there was a great balance to the challenges: An android application, web apps, Unix command breakouts, binary exploitation, etc.

Anyways, if you made it this far, let me know if you have any questions! I really enjoyed playing the challenge and writing this massive post, and I hope somebody got something out of it. Also, if you found a different/better way to solve these let me know that too!

-ropnop
