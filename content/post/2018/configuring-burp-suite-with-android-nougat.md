---
title: "Configuring Burp Suite With Android Nougat"
author: "ropnop"
date: 2018-01-18
summary: "Android Nougat changed the default behavior for apps, so installing the Burp CA to user certs no longer works. Here's two ways to bypass it"
toc: true
share_img: "/images/2018/01/burp_alerts-1.png"
tags: ["android","burp","nougat"]
---

This last weekend I started testing a new Android app for fun, and ran into some trouble getting Burp Suite working properly. I burned a whole afternoon troubleshooting the issue, and decided to write up what I found out and two different ways I got it working.

# Background
I've done quite a bit of Android testing in the past and my setup usually involves a [Genymotion](https://www.genymotion.com/) VM or my old rooted Nexus Tablet. I run Burp Suite locally, install the User Cert as outlined in Portswigger's [documentation](https://support.portswigger.net/customer/portal/articles/1841102-installing-burp-s-ca-certificate-in-an-android-device), configure a WiFi proxy and I'm off the races.

This particular app I wanted to test, however, required a minimum API level 24 (Android 7.0 - "Nougat") and suddenly it wasn't working. I followed the steps I always do but saw nothing but "connection reset" errors in Burp:

![Burp Alrets](/images/2018/01/burp_alerts.png)

After a few frustrating hours of troubleshooting, I finally figured out the issue lied with the latest versions of Android (API >= 24). Before I go any further, all the information I needed was found in these great write-ups:

 * https://serializethoughts.com/2016/09/10/905/
 * https://android-developers.googleblog.com/2016/07/changes-to-trusted-certificate.html

Starting with Nougat, Android changed the default behavior of trusting user installed certificates. It's no longer possible to just install the Burp CA from the sdcard to start intercepting app traffic. Unless otherwise specified, apps will now *only* trust system level CAs. The failure happens "invisibly" and is responsible for all the alerts I saw in Burp Suite.

There's two ways to bypass this, and I'll walk through them both.

 * **Install the Burp CA as a system-level CA on the device**. My recommendation for the easiest solution, but does require a rooted device. Also added benefit of not having to set a lockscreen PIN :)
 * **Modify the manifest and repackage the app**. Slightly more work, but doesn't require root privileges. 

*Note: I did all this with Burp Suite Pro on my Windows 10 machine and am using an Android 7.1 (API25) Genymotion VM, but the steps should be applicable to any setup.*

# Install Burp CA as a system-level trusted CA
Since the "traditional" way of installing a user certificate doesn't work anymore in Nougat and above, for me the easiest solution is to install the Burp CA to the system trusted certificates. You can see all the system CAs that are bundled with an Android device by going to *Settings -> Security -> Trusted Credentials* and viewing system CAs. You'll see the similar CAs you'd see in a browser bundle.

Trusted CAs for Android are stored in a special format in `/system/etc/security/cacerts`. If we have root privileges, it's possible to write to this location and drop in the Burp CA (after some modification).

__Export and convert the Burp CA__
The first step is to get the Burp CA in the right format. Using Burp Suite, export the CA Certificate in DER format. I saved it as `cacert.der`

![Export Burp CA](/images/2018/01/export_burp_ca.png)

Android wants the certificate to be in PEM format, and to have the filename equal to the `subject_hash_old` value appended with `.0`. 

*Note: if you are using OpenSSL <1.0, it's actually just the `subject_hash`, not the "old" one*

Use `openssl` to convert DER to PEM, then output the `subject_hash_old` and rename the file:

```
openssl x509 -inform DER -in cacert.der -out cacert.pem
openssl x509 -inform PEM -subject_hash_old -in cacert.pem |head -1
mv cacert.pem <hash>.0
```

For example, with my certificate:

![Converting Cert](/images/2018/01/openssl_convert.png)

__Copy the certificate to the device__
We can use `adb` to copy the certificate over, but since it has to be copied to the `/system` filesystem, we have to remount it as writable. As root, this is easy with `adb remount`.

```
adb root
adb remount
adb push <cert>.0 /sdcard/
```

The just drop into a shell (`adb shell`) and move the file to `/system/etc/security/cacerts` and chmod it to 644:

```
mv /sdcard/<cert>.0 /system/etc/security/cacerts/
chmod 644 /system/etc/security/cacerts/<cert>.0
```

Lastly, we have to full reboot the device with either `adb reboot` or a power cycle.

![Copy over the cert](/images/2018/01/copy_cert.png)

After the device reboots, browsing to *Settings -> Security -> Trusted Credentials* should show the new "Portswigger CA" as a system trusted CA.

![Portswigger Trusted](/images/2018/01/portswigger_system_ca.png)

Now it's possible to set up the proxy and start intecepting any and all app traffic with Burp :)

# Mofiying and repackaging an app
If you don't have root or don't want to modify the system trusted certificates, you can install the Burp CA as a user cert and then modify the specific APK you want to MitM.

Starting with Nougat, apps will ignore user-installed certificates by default. This is evident by looking at `logcat` output when launching the app:

![No network security config](/images/2018/01/no_network_config.png)

Without a network security config, the app will *only* trust system CAs and will not honor the user installed Burp certificate.

To get around this, it involves:

 * Disassembling the APK
 * Adding a new XML resource to define a network security profile
 * Modifying AndroidManifest.xml
 * Repackaging and self-signing the APK

__Disassemble and modify the APK__
Start by using [apktool](https://ibotpeaches.github.io/Apktool/) to disassemble the APK

```
apktool d TestApp.apk
```
![Apktool disassemble](/images/2018/01/apktool_d.png)

Next, add a new network security config by creating the file `network_security_config.xml` in the `res/xml` directory:

```
vim TestApp\res\xml\network_security_config.xml
```

The config needs to explicitly state that trusting user certs is acceptable. The entire contents should be:

```xml
<network-security-config> 
    <base-config> 
        <trust-anchors> 
            <!-- Trust preinstalled CAs --> 
            <certificates src="system" /> 
            <!-- Additionally trust user added CAs --> 
            <certificates src="user" /> 
        </trust-anchors> 
    </base-config> 
</network-security-config>
```
Finally, we have to define the network security config in `AndroidManifest.xml`. In the `<application>` tag, add the `android:networkSecurityConfig` attribute pointing to the new XML file:

```xml
<application android:allowBackup="true" android:networkSecurityConfig="@xml/network_security_config" ...etc...>
```

__Reassemble and Sign__
Finally, the APK must now be rebuilt and signed in order to be installed. Using `apktool b`, a new build will be created in the `dist/` directory:

```
apktool b TestApp
```

To self-sign the app, use `keytool` to create a new keystore and key, then `jarsigner` to sign the new APK:

```
keytool -genkey -v -keystore test.keystore -storepass password -alias android -keypass password -keyalg RSA -keysize 2048 -validity 10000


jarsigner.exe -verbose -keystore test.keystore -storepass password -keypass password TestApp\dist\TestApp.apk android
```

![Build and Sign APK](/images/2018/01/build_and_sign.png)

Lastly, install the new APK with `adb`:

```
adb install TestApp\dist\TestApp.apk
```

Now, when we start the application, the logcat output will indicate a new network security config is being used:

![Logcat new network security config](/images/2018/01/logcat_new_config.png)

With the Burp CA installed to user certificates, we can now MitM the application traffic!


# tl;dr cheatsheet
**Install System CA**
```
# Convert DER to PEM
openssl x509 -inform DER -in cacert.der -out cacert.pem

# Get subject_hash_old (or subject_hash if OpenSSL < 1.0)
openssl x509 -inform PEM -subject_hash_old -in cacert.pem |head -1

# Rename cacert.pem to <hash>.0
mv cacert.pem 9a5ba575.0

# Remount and copy cert to device
adb root
adb remount
adb push 9a5ba575.0 /sdcard/
adb shell
vbox86p:/ # mv /sdcard/9a5ba575.0 /system/etc/security/cacerts/
vbox86p:/ # chmod 644 /system/etc/security/cacerts/9a5ba575.0
vbox86p:/ # reboot
```

**Modify APK**
```
apktool d TestApp.apk
vim TestApp\res\xml\network_security_config.xml
#Content:
        <network-security-config> 
        <base-config> 
        <trust-anchors> 
        <!-- Trust preinstalled CAs --> 
        <certificates src="system" /> 
        <!-- Additionally trust user added CAs --> 
        <certificates src="user" /> 
        </trust-anchors> 
        </base-config> 
        </network-security-config>

vim TestApp\AndroidManifest.xml
# Add to <application > tag:
        android:networkSecurityConfig="@xml/network_security_config"

# Rebuild and self-sign
keytool -genkey -v -keystore test.keystore -storepass password -alias android -keypass password -keyalg RSA -keysize 2048 -validity 10000

apktool b TestApp

jarsigner -keystore test.keystore -storepass password -keypass password TestApp\dist\TestApp.apk android

# Install new APK
adb install TestApp\dist\TestApp.apk

# Install Burp CA to User Certs
mv cacert.der cacert.cer
adb push burpca.cer /mnt/sdcard
Settings -> Security -> Install from SD Card
```

Hope this helps!
-ropnop
