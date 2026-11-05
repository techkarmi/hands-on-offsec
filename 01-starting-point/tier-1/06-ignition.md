# Ignition

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Virtual host, web-app fingerprinting, weak admin credentials |

## Overview
A Magento store reachable via a virtual host, protected only by a weak admin password.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>       # 80 http
# the raw IP redirects to a .htb name -> add it to /etc/hosts
echo "<TARGET_IP> ignition.htb" | sudo tee -a /etc/hosts
# fingerprint: it's Magento -> default admin path /admin
```

## Exploitation
Logged into `http://ignition.htb/admin` with `admin:qwerty123`. The password matches
Magento's policy (letters + numbers, 7+ chars), so the laziest compliant leaked
password works; a hint of `******123` (6 hidden + 123) confirmed the length.

## Key Takeaway
Enforce strong admin passwords, rate limiting and account lockout; rename/protect
default admin paths.

## My note
Weak-credential guessing should match the application's password policy, not just throw
`admin@123`. And mind lockout: Magento (like AD and most apps) locks accounts after a
handful of failed attempts, so blind spraying can lock you out or, in the real world,
lock a real user and alert defenders.
