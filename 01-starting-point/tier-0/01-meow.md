# Meow

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Telnet, blank credentials |

## Overview
A Linux host exposing Telnet with no authentication hardening. The lesson is why
legacy clear-text remote access and blank accounts are indefensible.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
# 23/tcp open  telnet
```

## Exploitation
```bash
telnet <TARGET_IP>
# login: root   (no password)
```
Logged in directly as **root** with a blank password.

## Key Takeaway
Never expose Telnet; use SSH. Disable blank/default accounts and enforce strong auth.

## My note
This was my first box. Early on I made the classic mistake of scanning my own VPN IP
(10.10.x.x) instead of the target (10.129.x.x). Internalising that distinction
immediately is the first real skill of the platform.
