# Crocodile

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Credentials in anonymous FTP, web admin login |

## Overview
Anonymous FTP leaks credential files that unlock a hidden web admin panel. A clean
example of chaining two weak services.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>       # 21 ftp, 80 http
ftp <TARGET_IP>                              # anonymous
ftp> get allowed.userlist
ftp> get allowed.userlist.passwd
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt -x php
# /login.php
```

## Exploitation
Paired the usernames and passwords recovered from FTP and authenticated at
`/login.php`.

## Key Takeaway
Never store credential files on anonymous shares; separate and harden admin interfaces.
