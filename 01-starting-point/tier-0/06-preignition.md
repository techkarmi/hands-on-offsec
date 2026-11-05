# Preignition

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Directory brute force, default web credentials |

## Overview
A web server hiding an admin panel discoverable through content discovery.

## Enumeration
```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt -x php
# /admin.php (200)
```

## Exploitation
Browsed to `/admin.php` and authenticated with default credentials `admin:admin`.

## Key Takeaway
Remove default credentials and do not expose admin panels without hardening and rate
limiting.
