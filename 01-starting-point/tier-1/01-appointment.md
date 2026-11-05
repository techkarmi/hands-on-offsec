# Appointment

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | SQL injection authentication bypass |

## Overview
A web login form that builds its SQL query from raw input, allowing an authentication
bypass via SQL injection.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>       # 80/tcp http
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt
```
Found a login form.

## Exploitation
First I confirmed injectability by hand: a single quote broke the query, and
`' OR '1'='1` behaved differently from `' OR '1'='2`. Then the auth bypass:
```
Username: admin' -- -
Password: anything
```
The `-- -` comments out the rest of the query (including the password check), so the
backend only evaluates `WHERE user = 'admin'` and logs me in.

## Key Takeaway
Use parameterised queries / prepared statements. Never concatenate user input into SQL.

## My note
Testing `'` then `' OR '1'='1` vs `' OR '1'='2` is my standard manual SQLi detection
before reaching for sqlmap. Understanding why `#` (MySQL) or `-- -` terminates the
query is what lets you adapt the payload when the obvious one is filtered.
