---
title: "Leviathan CTF"
excerpt_separator: "<!--more-->"
categories:
  - CTF
---

![center-aligned-image](https://cdn.pixabay.com/photo/2012/05/02/19/01/whale-46013_1280.png){: .align-center}

CTF by **OverTheWire** @ [https://overthewire.org/wargames/leviathan/](https://overthewire.org/wargames/leviathan/)
{: .notice--info}

As per the official description: this wargame doesn't require any knowledge about programming - just a bit of common sense and some knowledge about basic *nix commands.

<!--more-->

| Content: | 
|-|-|-|-|
| [Install](#install) | [Leviathan 00](#leviathan-00) | [Leviathan 04](#leviathan-04) | 
|-|-|-|-|
| | [Leviathan 01](#leviathan-01) | [Leviathan 05](#leviathan-05) | 
|-|-|-|-|
| | [Leviathan 02](#leviathan-02) | [Leviathan 06](#leviathan-06) |
|-|-|-|-|
| | [Leviathan 03](#leviathan-03) | [Leviathan 07](#leviathan-07) |
|-|-|-|-|


# Capture the Flag

### [Install]
All the challenges can be accessed with ssh. The syntax is the following: \
`ssh leviathanX@leviathan.labs.overthewire.org -p 2223` \
All the flags are in `/etc/leviathan_pass/`, so those are the files we should aim to read in each level. Obviously, we need to gain the right permissions first. \
In the machine there are a couple of tools already installed, in particular PEDA: this is a gdb extension that shows registers, assembly and stack automatically, reducing a lot of the usual pain when using gdb. To start it, run gdb as usual and then source to the peda.py file:

```bash
gdb --args filetoinspect param1 param2
> source /usr/local/peda/peda.py
```

### [Leviathan 00]
`ssh leviathan0@leviathan.labs.overthewire.org -p 2223` \
Password: leviathan0

The home folder contains a hidden folder, which contains a single file. We look into it to see if it contains anything labelled as flag for this level.

```bash
ls -lah
cd .backup/
ls -alh
bookmarks.html | grep leviathan
```

### [Leviathan 01]
`ssh leviathan1@leviathan.labs.overthewire.org -p 2223` \
The home folder contains an executable with suid, meaning (in a really basic summary) that it will run with the permissions of the creator and not of the user. We can see that the creator has permissions to see the password file that we need, so we have to execute this file. If we give the right password, we will have a shell with proper permissions. \
We could use gdb to step through the execution, but the ltrace command shows immediately what we need, so we will take the easy way. After seeing the password from the string compare, we can run the program on its own and get a shell from which we can cat the file.

> ltrace: this program runs the specified command and intercepts all library and system calls.

```bash
ls -lah
ltrace ./check [insert random password when prompted]
> strcmp("AAA", "sex") 
./check [insert password 'sex' when prompted]
cat /etc/leviathan_pass/leviathan2
```

### [Leviathan 02]
`ssh leviathan2@leviathan.labs.overthewire.org -p 2223` \
This challenge is based on a little bug in how the `printfile` executable works. The bug can be spotted after playing around a bit with filenames passed to the executable: `ltrace`-ing shows how when passing a file with a space in its name, only the word before the space is used in the `access()` call. \
When trying to access the password file directly, we are warned that we can't access it. But we can exploit the bug by creating a symlink to that file called 'password', and then creating a file with the name 'password bug'. When trying to access 'password bug' with the executable, it will check permissions on 'password bug' without triggering the warning, and then go on and print 'password'.

```bash
ls -lah
ltrace ./printfile name space
> access("name", 4)
ln -s /etc/leviathan_pass/leviathan3 '/tmp/password'
touch '/tmp/password bug'
./printfile '/tmp/password bug'
```

### [Leviathan 03]
`ssh leviathan3@leviathan.labs.overthewire.org -p 2223` \
This challenge is really similar to the one in level 01. The only difference is that there is another string compare to try and confuse us.

```bash
ls -lah
ltrace ./level3 [insert random password when prompted]
> strcmp("AAAA\n", "snlprintf\n")  
./level3 [insert password 'snlprintf' when prompted]
cat /etc/leviathan_pass/leviathan4
```

### [Leviathan 04]
`ssh leviathan4@leviathan.labs.overthewire.org -p 2223` \
The flag in this challenge is not hidden, but encoded. After executing the file, we get a string of binary numbers. \
Checking their values we get hex numbers within the ASCII range, and from there on it's just a matter of looking up the ASCII table.

```bash
ls -lah
cd .trash
./bin
> 01010100 01101001 01110100 01101000 00110100 01100011 01101111 01101011 01100101 01101001 00001010
> 0x54 0x69 0x74 0x68 0x34 0x63 0x6F 0x6B 0x65 0x69 0x0A
```

### [Leviathan 05]
`ssh leviathan5@leviathan.labs.overthewire.org -p 2223` \
When executing the file with ltrace, we can see that it is trying to open a file that does not exist. The only thing we need to do is create that file, with the content that we want, and to do that we use a symlink.

```bash
ls -lah
ltrace ./leviathan5
> fopen("/tmp/file.log", "r")
ln -s /etc/leviathan_pass/leviathan6 /tmp/file.log
./leviathan5
```

### [Leviathan 06]
`ssh leviathan6@leviathan.labs.overthewire.org -p 2223` \
In this challenge `ltrace` is not enough because the compare is done with a value in memory, and not between strings. The `atoi()` method just reads the input string and casts it to integer, which is stored in eax as hex value after the method returns. \
A few instructions after we have a compare between that value and a value stored in memory, which is our password. We only have to give that password as input, in decimal format.

```bash
ls -lah
gdb --args leviathan6 1111
> atoi(4444) -> eax = 0x1BD3
> cmp eax, 0x1BD3
```

### [Level 07]
`ssh leviathan7@leviathan.labs.overthewire.org -p 2223` \
Congratulations, no more challenges. Leviathan is complete. 