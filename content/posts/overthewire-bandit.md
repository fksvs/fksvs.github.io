+++
date = '2025-12-13T16:46:20+03:00'
title = 'OverTheWire Bandit Solutions'
description = 'Solutions for Bandit Game from OverTheWire'
categories = ['overwthewire', 'linux', 'cybersecurity']
+++

I recently started the OverTheWire wargames to sharpen my Linux skills. Here is a log of the steps I took, the challenges I faced, and the commands I used to solve the Bandit levels.<!--more-->

## Level 0 > Level 1

```
$ ssh bandit0@bandit.labs.overthewire.org -p 2220
$ ls
readme
$ cat readme
ZjLjTmM6FvvyRnrb2rfNWOZOTa6ip5If
```

## Level 1 > Level 2

```
$ ssh bandit1@bandit.labs.overthewire.org -p 2220
$ cat ./"-"
263JGJPfgU6LtdEvgfWU1XP5yac29mFx
```

You should use quotes or backslash to access files or directories with space in its name.
## Level 2 > Level 3

```
$ ssh bandit2@bandit.labs.overthewire.org -p 2220
$ cat "./--spaces in this filename--"
MNk8KNH3Usiio41PRUEoDFPqfxLPlSmx
```

## Level 3 > Level 4

```
$ ssh bandit3@bandit.labs.overthewire.org -p 2220
$ ls -al
total 24  
drwxr-xr-x   3 root root 4096 Oct 14 09:26 .  
drwxr-xr-x 150 root root 4096 Oct 14 09:29 ..  
-rw-r--r--   1 root root  220 Mar 31  2024 .bash_logout  
-rw-r--r--   1 root root 3851 Oct 14 09:19 .bashrc  
drwxr-xr-x   2 root root 4096 Oct 14 09:26 inhere  
-rw-r--r--   1 root root  807 Mar 31  2024 .profile
$ cd inhere
$ ls -al
total 12  
drwxr-xr-x 2 root    root    4096 Oct 14 09:26 .  
drwxr-xr-x 3 root    root    4096 Oct 14 09:26 ..  
-rw-r----- 1 bandit4 bandit3   33 Oct 14 09:26 ...Hiding-From-You
$ cat ...Hiding-From-You
2WmrDFRmJIq3IPxneAaMGhap0pFhF3NJ
```

## Level 4 > Level 5

```
$ ssh bandit4@bandit.labs.overthewire.org -p 2220
$ cd inhere
$ file ./"-file07"
./-file07: ASCII text
$ cat ./"-file07"
4oQYVPkxZOOEOO5pTW81FB8j8lxXGUQw
```

## Level 5 > Level 6

```
$ ssh bandit5@bandit.labs.overthewire.org -p 2220
$ cd inhere
$ find . -type f -name "*" -size 1033c
./maybehere07/.file2
$ cat maybehere07/.file2
HWasnPhtq9AVKe0dmk45nxy20cvUa6EG
```

## Level 6 > Level 7

```
$ ssh bandit6@bandit.labs.overthewire.org -p 2220
$ find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
/var/lib/dpkg/info/bandit7.password
$ cat /var/lib/dpkg/info/bandit7.password
/var/lib/dpkg/info/bandit7.password
```

## Level 7 > Level 8

```
$ ssh bandit7@bandit.labs.overthewire.org -p 2220
$ cat data.txt | grep millionth
millionth       dfwvzFQi4mU0wfNbFOe9RoWskMLg7eEc
```

## Level 8 > Level 9

```
$ ssh bandit8@bandit.labs.overthewire.org -p 2220
$ cat data.txt | sort | uniq -u
4CKMh1JI91bUIZZPXDqGanal4xvAg0JM
```

## Level 9 > Level 10

```
$ ssh bandit9@bandit.labs.overthewire.org -p 2220
$ strings data.txt | sort | grep ==
5========== FGUW5ilLVJrxX9kMYMmlN4MgbpfMiqey  
E========== is  
========== password  
========== the
```

## Level 10 > Level 11

```
$ ssh bandit10@bandit.labs.overthewire.org -p 2220
$ base64 -d data.txt
The password is dtR173fZKb0RRsDFSGsg2RWnpNVj3qRr
```

## Level 11 > Level 12

```
$ ssh bandit11@bandit.labs.overthewire.org -p 2220
$ cat data.txt
Gur cnffjbeq vf 7k16JArUVv5LxVuJfsSVdbbtaHGlw9D4
```

Copy this data, and go to www.rot13.com:

`The password is 7x16WNeHIi5YkIhWsfFIqoognUTyj9Q4`
## Level 12 > Level 13

```
$ ssh bandit12@bandit.labs.overthewire.org -p 2220
$ mktemp -d
/tmp/tmp.HqW7SzZNqZ
$ mv data.txt /tmp/tmp.HqW7SzZNqZ
$ cd /tmp/tmp.HqW7SzZNqZ
$ xxd -r data.txt > data_text.txt
$ file data_text.txt
data_text.txt: gzip compressed data, was "data2.bin", last modified: Tue Oct 14 09:26:00 2025, max compr  
ession, from Unix, original size modulo 2^32 572
$ mv data_text.txt data.gz
$ gzip2 -d data.gz
$ file data
data: bzip2 compressed data, block size = 900k
$ mv data data.bz
$ bzip2 -d data.bz
$ file data
data: gzip compressed data, was "data4.bin", last modified: Tue Oct 14 09:26:00 2025, max compressi  
on, from Unix, original size modulo 2^32 20480
$ mv data data.gz
$ gzip -d data.gz
$ file data
data: POSIX tar archive (GNU)
$ tar -xvf data
data5.bin
$ file data5.bin
data5.bin: POSIX tar archive (GNU)
$ tar -xvf data5.bin
data6.bin
$ file data6.bin
data6.bin: bzip2 compressed data, block size = 900k
$ mv data6.bin data6.bz
$ bzip2 -d data6.bz
$ file data6
data6: POSIX tar archive (GNU)
$ tar -xvf data6
data8.bin
$ file data8.bin
data8.bin: gzip compressed data, was "data9.bin", last modified: Tue Oct 14 09:26:00 2025, max comp  
ression, from Unix, original size modulo 2^32 49
$ mv data8.bin data8.gz
$ gzip -d data8.gz
$ file data8
data8: ASCII text
$ cat data
The password is FO5dwFsc0cbaIiH0h8J2eUks2vdTDwAn8
```
