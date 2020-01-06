---
title: "Cracking OpenNMS Password Hashes"
author: "ropnop"
draft: true
date: 2017-06-21
summary: "After compromising an OpenNMS server, I recovered salted password hashes. I couldn't find any info online, so I reversed them and wrote a tool to crack them"
toc: true
share_img: "/images/2017/06/opennms.jpeg"
tags: ["hash", "password", "opennms", "python"]
---

## Background
On my last internal penetration test, I compromised a server running [OpenNMS](https://www.opennms.org/en). During post-exploitation, I recovered several password hashes for local OpenNMS users, but for the life of me could not figure out how they were hashed. Even though I had root access on the server, I still wanted to try to crack these hashes since I figured maybe the users were re-using their passwords elsewhere in the environment.

My Googling failed me and I didn't find any good resources online about OpenNMS password hashes, so I decided to document what I found out and release a Python tool to hopefully help any future pentesters who compromise an OpenNMS server.

**tl;dr**. They're base64 encoded strings with the first 16 bytes being the salt, and the remaining 32 bytes being 100,000 iterations of `sha256(salt.password)`. Tool released here:

[https://github.com/ropnop/opennms\_hash\_cracker](https://github.com/ropnop/opennms_hash_cracker)

## Initial Compromise

We discovered a server running an outdated version of [OpenNMS](https://www.opennms.org/en) with a known Java deserialization vulnerability. FoxGlove Security released an amazing writeup digging into this vulnerability here:

https://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/

Exploitation couldn't be easier with the [Metasploit module](https://www.rapid7.com/db/modules/exploit/linux/misc/opennms_java_serialize). We pointed the module to the RMI interface on port 1099 and set the payload to be `linux/x86/shell_reverse_tcp` and were quickly granted a shell running with root privileges:

![OpenNMS Session](/images/2017/06/opnenms_shell.png)

OpenNMS was installed in `/opt/opennms` so I used the shell to explore that directory and quickly found a file that defined local user accounts for OpenNMS. I recognized some of the usernames as privileged administrators of the organization, so I wanted to try to crack some of these hashes in the hope that the passwords were reused elsewhere.

The hashes were stored in `/opt/opennms/etc/users.xml` and looked like this:

```xml
<?xml version="1.0"?>
<userinfo xmlns="http://xmlns.opennms.org/xsd/users">
  <header>
    <rev>.9</rev>
    <created>Wednesday, February 6, 2002 10:10:00 AM EST</created>
    <mstation>master.nmanage.com</mstation>
  </header>
<users>
<user>
      <user-id>ropnop</user-id>
      <full-name>Rop Nop</full-name>
      <user-comments></user-comments>
      <password salt="true">L5j2hiVX4B+LTHnY3Mwq5m5dBZzNdHhiBpvBjyCerBsBqqJcxRUsRAxaDQtjRkcn</password>
      <role>ROLE_RTC</role>
    </user>

...(more users)...

  </users>
</userinfo>
```
Obviously I was interested in the password hash, which according to the XML was salted:
`L5j2hiVX4B+LTHnY3Mwq5m5dBZzNdHhiBpvBjyCerBsBqqJcxRUsRAxaDQtjRkcn`

I quickly searched through both `john` and `hashcat` help pages to see if there was a mode for OpenNMS hashes, but found nothing. I then resorted to Google, but still found no hints on how the hash was salted and/or calculated.

At this point I realized I was going to have to figure it out on my own.

## Hash Identification
Most password cracking programs use hex representations of hashes, so I converted the base64 value in the XML to hex:

```python
>>> "L5j2hiVX4B+LTHnY3Mwq5m5dBZzNdHhiBpvBjyCerBsBqqJcxRUsRAxaDQtjRkcn".decode('base64').encode('hex')
'2f98f6862557e01f8b4c79d8dccc2ae66e5d059ccd747862069bc18f209eac1b01aaa25cc5152c440c5a0d0b63464727'
```

Next I used the Python package [hashID](https://pypi.python.org/pypi/hashID) to see possible hashes:

![HashID](/images/2017/06/hashid.png)

So it's potentially SHA-384, but that's so rare I really doubt it. Time to dig deeper.

### Plaintext Identification
Even if I correctly figured out the hashing algorithm used, I still wasn't sure how the salt was implemented, let alone what the salt actually was. My first thought was that maybe the salt was stored in the PostgresSQL database used by OpenNMS. Since I had root on the server, I was able to connect to the database and explore the tables, but found nothing related to passwords or salts. So it had to be in the application somewhere.

OpenNMS is open source, so the first place I turned was to its Github page and just did a [search for "salt"](https://github.com/OpenNMS/opennms/search?utf8=%E2%9C%93&q=salt&type=). That turned up some example `users.xml` files used for [test purposes](https://github.com/OpenNMS/opennms/blob/2b2ed9a50a88e9ce898842784ad3fcf2f6d1ae3f/features/springframework-security/src/test/resources/org/opennms/web/springframework/security/users.xml).

Following the source code up from there, I finally ran across some [assertion tests]( https://github.com/OpenNMS/opennms/blob/2b2ed9a50a88e9ce898842784ad3fcf2f6d1ae3f/features/springframework-security/src/test/java/org/opennms/web/springframework/security/AuthenticationIT.java) that used the salted password hashes:

```java
@Test
    public void testAuthenticateRtc() {
        org.springframework.security.core.Authentication authentication = new UsernamePasswordAuthenticationToken("rtc", "rtc");
        org.springframework.security.core.Authentication authenticated = m_provider.authenticate(authentication);
        assertNotNull("authenticated Authentication object not null", authenticated);
        Collection<? extends GrantedAuthority> authorities = authenticated.getAuthorities();
        assertNotNull("GrantedAuthorities should not be null", authorities);
        assertEquals("GrantedAuthorities size", 1, authorities.size());
        assertContainsAuthority(Authentication.ROLE_RTC, authorities);
    }
```

Matching that up with the "rtc" user hash in the test `users.xml` I now had a matching plaintext and hash:

* **Plaintext:** `rtc`
* **Salted Hash:** `L5j2hiVX4B+LTHnY3Mwq5m5dBZzNdHhiBpvBjyCerBsBqqJcxRUsRAxaDQtjRkcn`

It still didn't tell me what the salt was or how it was used, but having a positive test case was important in figuring out the hashing scheme.

### Hashing Algorithm Identification
Instead of focusing on the Github code, I turned back to my root shell on the server. I wasn't sure how out of date or how different the code might be between Github and what I was dealing with, so I wanted to try to find the source code on the server where the hash was calculated.

From within the `opennms` directory, I simply ran a search for the string "salt" to see where it was used:

```bash
$ find . -type f -exec grep -i -l salt {} \;
./etc/users.xml
./jetty-webapps/opennms/includes/angular.js
./jetty-webapps/opennms/RemotePollerMap/openlayers/lib/OpenLayers/Layer/HTTPRequest.js
./jetty-webapps/opennms/RemotePollerMap/openlayers/lib/OpenLayers/Tile/Image/IFrame.js
./jetty-webapps/opennms/RemotePollerMap/openlayers/OpenLayers.js
./jetty-webapps/opennms-remoting/webstart/jasypt-1.9.0.jar
./jetty-webapps/opennms-remoting/webstart/snmp4j-1.11.1.jar
./jetty-webapps/opennms-remoting/webstart/spring-security-config-3.1.0.RELEASE.jar
./jetty-webapps/opennms-remoting/webstart/spring-security-core-3.1.0.RELEASE.jar
./lib/bcprov-jdk14-1.38.jar
./lib/jasypt-1.9.0.jar
./lib/joda-time-2.1.jar
./lib/snmp4j-1.11.1.jar
./lib/spring-security-config-3.1.0.RELEASE.jar
./lib/spring-security-core-3.1.0.RELEASE.jar
./system/joda-time/joda-time/1.6.2/joda-time-1.6.2.jar
./system/org/apache/servicemix/bundles/org.apache.servicemix.bundles.jasypt/1.6_1/org.apache.servicemix.bundles.jasypt-1.6_1.jar
```
Alright, so there's some JAR libraries that have the word "salt" in them, in particular `./lib/jasypt-1.9.0.jar`

According to their homepage:
> Jasypt is a java library which allows the developer to add basic encryption capabilities to his/her projects with minimum effort, and without the need of having deep knowledge on how cryptography works.

Digging into the docs, I finally came across what looked pretty promising: a class called [StrongPasswordEncryptor](http://www.jasypt.org/api/jasypt/1.8/org/jasypt/util/password/StrongPasswordEncryptor.html), which uses another class, called [StandardStringDigester](http://www.jasypt.org/api/jasypt/1.8/org/jasypt/digest/StandardStringDigester.html). According to the docs, StrongPasswordEncryptor is a:

![Salt Size info](/images/2017/06/salt_size.png)

Jasypt is also on Github, and I found the source code for both classes [here](https://github.com/jboss-fuse/jasypt/blob/master/jasypt/src/main/java/org/jasypt/util/password/StrongPasswordEncryptor.java) and [here](https://github.com/jboss-fuse/jasypt/blob/a6cbf5419bc153698368aa796d619e67bb197ba9/jasypt/src/main/java/org/jasypt/digest/StandardStringDigester.java). Looking in the comments of StandardStringDigester source code, I finally have my answer on how the digest is constructed:

![Digest Construction](/images/2017/06/create_digest.png)

I have all the pieces I need now (hopefully):

* **Plaintext:** "rtc"
* **Digest:** L5j2hiVX4B+LTHnY3Mwq5m5dBZzNdHhiBpvBjyCerBsBqqJcxRUsRAxaDQtjRkcn
* **Salt Size:** 16 bytes
* **Digest Format:** (salt.password)
* **Algorithm:** sha256
* **Iterations:** 100,000

## Putting it all together
I really wanted to verify this was the correct algorithm and that I could in fact derive the digest from the plaintext

Now to test it, I needed to concatenate the bytes of the salt with the plaintext, then calculate a sha256 digest 100,000 times. I wrote a Python script to verify the plaintext and password:

<script src="https://gist.github.com/ropnop/abb60daba012548f7429a288394a23dd.js"></script>

And testing it with the known plaintext we have success with 100,000 iterations!

![Plaintext Check](/images/2017/06/plaintext_check.png)

### Cracking with a wordlist
Now that I had a PoC working, I wanted to throw a wordlist at the hashes I had compromised. Since it was just a sha256 hash with a salt, I thought I could use `john` or `hashcat`. I asked on [Twitter](https://twitter.com/ropnop/status/875802857882112001) if it was possible to specify number of rounds, but unfortunately found an answer that worked. [@coffeetocode](https://twitter.com/coffeetocode) dug around in John's source code and found that sha256crypt was probably the closest I could get. The problem is that that sha256crypt has a 16 character max for the salt, and these use 16 *byte* salts.

Looked like I was out of luck with using a cracking program. If anyone has any suggestions or workarounds though, please let me know!

### Writing a cracker
Since I still wanted to run some plaintext passwords at the hashes, and I was totally [nerd-sniped](https://xkcd.com/356/) by this whole ordeal, I wrote a simple brute-forcer in Python.

It's on my Github here:

[https://github.com/ropnop/opennms\_hash\_cracker](https://github.com/ropnop/opennms_hash_cracker)

The Python script ingests a `users.xml` file, extracts the hashes, and then runs a cracking attack against the hashes using a supplied wordlist.

It also has some logic to parse and test unsalted hashes for OpenNMS as well (which are just MD5 digests).

Running it on the example `users.xml` file from the OpenNMS [test data](https://github.com/OpenNMS/opennms/blob/2b2ed9a50a88e9ce898842784ad3fcf2f6d1ae3f/features/springframework-security/src/test/resources/org/opennms/web/springframework/security/users.xml) with a simple wordlist shows it works:

![Cracking hashes](/images/2017/06/cracking_hashes.png)

It's not very fast or efficient, but it should serve the purpose if you have a small wordlist or just want to try some common passwords. Would love some help in speeding it up or maybe even porting it to hashcat!

In the end, this didn't actually help me at all on my pentest as I was low on time and had to pursue other avenues, but it was fun exercise nonetheless. Hopefully this blog and tool will help some future pentester down the line if he or she ever compromises an OpenNMS box.

Enjoy!
-ropnop
