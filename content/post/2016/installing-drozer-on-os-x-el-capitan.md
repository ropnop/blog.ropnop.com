---
title: "Installing Drozer on OS X El Capitan"
date: 2016-04-14
toc: true
tags: ["android", "osx"]
author: "ropnop"
summary: "Turns out installing Drozer on OS X is a pain. The latest versions of OS X doesn't include OpenSSL headers, which breaks dependencies. Here's my workaround."
share_img: "/images/2016/04/drozer.png"
---

## Background
I like to use OS X for as much as possible when doing assessments, but on a recent Android test I had a hell of a time getting Drozer installed. I finally got it working and figured I'd share the steps in case anyone else out there runs into the same problems I had.

If you're not familiar, Drozer is an awesome attack and testing framework for Android. You can read more about it here: https://labs.mwrinfosecurity.com/tools/drozer/ or on its main GitHub page https://github.com/mwrlabs/drozer

It's written in Python and you can download the egg file. Since I do most of my android testing from Mac (with Genymotion and Burp), I figured I would just download the Drozer egg and install it with my native Python. The [installation instructions](https://github.com/mwrlabs/drozer/blob/develop/INSTALLING) don't mention OS X, but they seemed simple enough for Linux (just using easy_install). 

Turns out there's quite a few dependency issues which were a pain to resolve. I'm gonna walk through the steps I took to finally get it up and running.

## Dependency Hell
If you just download the egg and try to use "easy_install" on Mac, you'll get some spectacular error codes when trying to install the dependencies. Drozer requires the following python modules:
```bash
$ cat requirements.txt
cffi==1.1.2
cryptography==0.9.3
drozer==2.3.4
enum34==1.0.4
idna==2.0
ipaddress==1.0.14
protobuf==2.4.1
pyasn1==0.1.8
pycparser==2.14
pyOpenSSL==0.13
six==1.9.0
Twisted==10.2.0
zope.interface==4.1.2
```

The root of the problems is in pyOpenSSL. OS X El Capitan doesn't come with OpenSSL installed anymore, and trying to compile pyOpenSSL will fail because it can't find the required headers.

### Installing Dependencies
First of all, don't try to use Mac's built-in python. Install python with homebrew (`$ brew install python`) and make sure you can "pip install" modules without needing sudo. Secondly, since Drozer has some hard coded dependencies on specific versions, I opted to install everything in a python virtual environment. I prefer to use virtualenvwrapper to create and manger virtualenvs:
```bash
$ pip install virtualenvwrapper
$ mkvirtualenv drozer
$ workon drozer
```

**Install OpenSSL.** Use brew to install OpenSSL back onto OS X El Capitan. If it's already present, uninstall and reinstall it:
```bash
$ brew uninstall openssl #if installed already
$ brew install openssl
```

**Compile pyOpenSSL.** Unfortunately, Drozer requires a specific version of pyOpenSSL - and this version has a typo that prevents it from compiling successfully. This took me forever to figure out until I saw a similar error and the fix [here](https://github.com/sumanj/frankencert/issues/4). We need to download the source for pyOpenSSL v0.13 then fix the typo (using sed):
```bash
$ wget https://pypi.python.org/packages/source/p/pyOpenSSL/pyOpenSSL-0.13.tar.gz
$ tar xzvf pyOpenSSL-0.13.tar.gz
$ cd pyOpenSSL-0.13
$ sed -i '' 's/X509_REVOKED_dup/X509_REVOKED_dupe/' OpenSSL/crypto/crl.c
```
The sed command fixes the typo ('dup' vs 'dupe'). Next we need to build pyOpenSSL, but need to specify the location of the OpenSSL headers we installed from brew (using build_ext the '-L' and '-I' options):
```bash
$ python setup.py build_ext -L/usr/local/opt/openssl/lib -I/usr/local/opt/openssl/include
$ python setup.py build
$ python setup.py install
``` 
*Note: make sure you are in your drozer virtualenv before doing the build/install steps!*

**Install other dependencies.** Once pyOpenSSL v0.13 is installed in the virtualenv, use easy_install to install the other dependencies:
```bash
$ easy_install --allow-hosts pypi.python.org protobuf==2.4.1
$ easy_install twisted==10.2.0 #ignore any warnings/errors, it works
```

**Install Drozer**. Finally, download the latest Drozer egg file from here: https://www.mwrinfosecurity.com/products/drozer/. Still inside your drozer virtualenv, use easy_install to install it. If all goes right it should work with no errors:
```bash
$ easy_install ./drozer-2.3.4-py2.7.egg
```

### Running Drozer
You can now always run drozer from the virtual environment you created. Since I get sick of switching to virtual environments, it's possible to just create a shortcut to run drozer from the virtual environment it's in.

Create the file `/usr/local/bin/drozer` (or wherever you want the shortcut to be). For the shebang, put the absolute path the python executable for the drozer virtual environment. The file should look like this:
```python
#!/Users/rflather/.virtualenvs/drozer/bin/python
# EASY-INSTALL-SCRIPT: 'drozer==2.3.4','drozer'
__requires__ = 'drozer==2.3.4'
__import__('pkg_resources').run_script('drozer==2.3.4', 'drozer')
```

Now you have a command "drozer" in your path and can run it anywhere:

![Drozer running on OS X](/images/2016/04/drozer_running.png)

Hope this helps somebody! Happy android hacking :)

-ropnop