# Fawn

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Anonymous FTP |

## Overview
A host running FTP with anonymous access enabled.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
# 21/tcp open  ftp (vsftpd), anonymous login allowed
```

## Exploitation
```bash
ftp <TARGET_IP>          # user: anonymous, blank password
ftp> ls
ftp> binary             # so non-text files are not corrupted in transfer
ftp> get flag.txt
```

## Key Takeaway
Disable anonymous FTP; move to SFTP/FTPS with authentication. Anonymous file services
are a constant real-world data-exposure source.
