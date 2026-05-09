# Vulnversity — TryHackMe
## Info
- OS: Ubuntu
- Ports: 21, 22, 139, 445, 3128, 3333

## Enumeration
- gobuster dir -u http://target:3333 -w common.txt
- found /internal/ — file upload page

## Vulnerability
- unrestricted file upload
- .phtml extension bypasses filter

## Exploitation
- download php-reverse-shell.php from pentestmonkey
- rename to shell.phtml
- edit $ip and $port
- intercept upload with Burp
- fuzz extension with Intruder to find .phtml works
- nc -lvnp port
- visit http://target:3333/internal/uploads/shell.phtml

## Privilege Escalation
- find / -perm -u=s -type f 2>/dev/null
- /bin/systemctl has SUID bit
- exploit systemctl SUID to get root shell

## Lessons
- always fuzz file upload extensions
- SUID binaries are privesc goldmine
- GTFOBins for SUID exploitation reference
