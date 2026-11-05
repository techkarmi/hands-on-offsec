# Screenshots

Drop your own proof screenshots here, one folder per box, then reference them in the
writeup (`![alt](../images/<box>/NN-name.png)`). Screenshots are the single biggest
credibility upgrade for a portfolio: they prove you did the work rather than copied it.

**Golden rule:** crop out flag values. Show the proof of access (prompt, `id`,
`whoami`), not the flag string.

## Priority shot-list (do these first, they carry the portfolio)

### forest/  (flagship, highest priority)
- `01-rpcclient-enumdomusers.png` - null-session user list showing svc-alfresco
- `02-asrep-hash-and-crack.png` - the captured $krb5asrep$ hash and the cracked password
- `03-bloodhound-path.png` - the BloodHound shortest-path graph to Domain Admin (the money shot)
- `04-secretsdump-dcsync.png` - DCSync dumping the Administrator NTLM hash
- `05-pth-domain-admin.png` - SYSTEM shell on the DC via Pass-the-Hash (`whoami`)

### unified/
- `01-log4shell-callback.png` - tcpdump on 389 showing the target's JNDI callback
- `02-roguejndi-and-shell.png` - rogue-jndi serving the payload + the reverse shell landing

### oopsie/
- `01-cookie-tamper.png` - Burp showing role=admin cookie + the unlocked admin panel
- `02-path-hijack-root.png` - the SUID PATH hijack and `id` = uid=0(root)

### archetype/
- `01-xp-cmdshell-whoami.png` - mssqlclient xp_cmdshell returning sql_svc
- `02-psexec-system.png` - `whoami` = nt authority\system

### markup/
- `01-xxe-idrsa.png` - the XXE response leaking the SSH private key
- `02-scheduled-task-system.png` - the caught SYSTEM shell

### included/ , base/ , vaccine/
- one "proof of root" shot each: `id` showing uid=0(root) (crop the flag)

## Everything else (nice to have)
A single "proof of access" screenshot per Starting Point box is plenty. Do not pad the
easy Tier 0 boxes; one clean terminal shot is enough.
