# TryHackMe: Mr. Robot CTF Writeup

## Written by: Koushiq Murad

Hello friends! In this writeup, I'm documenting my process for successfully completing the **Mr. Robot** CTF room on TryHackMe. It was a fantastic challenge that took me through web enumeration, credential discovery, exploiting a weak WordPress configuration, and leveraging a SUID binary for the final flag.

---

## 1. Introduction and Initial Setup

My first steps, as always, were to connect to the TryHackMe VPN and confirm connectivity with the target machine.

1.  **Connectivity Check:** I started by sending a quick `ping` to make sure the target was alive and responding.
2.  **Initial Directory:** To keep everything organized, I created a dedicated directory for the CTF:
    ```bash
    mkdir robot
    cd robot
    ```

### Port Scanning with Nmap

Next, I launched a comprehensive Nmap scan to enumerate open ports, identify services, and grab version information (`-sV -sC`).

```bash
nmap -sV -sC -oA nmap.results <TARGET_IP>

My results showed:

    Port 22/tcp: ssh (closed - suggesting a firewall allows traffic but no service is listening)

    Port 80/tcp: http (Running an HTTP web server)

    Port 443/tcp: https (Running an HTTPS web server)

2. Web Enumeration and Initial Key Retrieval

Since Port 80 was open, the web server was my first point of attack. After checking out the cool, faux-terminal front page, I knew the real progress would come from finding hidden files and directories.

Directory Busting with GoBuster

I fired up GoBuster using a standard medium wordlist to quickly find interesting paths.
Bash

gobuster dir -u http://<TARGET_IP> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -o gobuster_results

By filtering the results for a 200 OK status code, I highlighted a few targets: /robots, /wp-login, and /license.

Key 1 and Credentials Discovery

    robots.txt (/robots): I immediately checked this file, and sure enough, it contained a path to the very first key!

        Key 1: The file key-1-of-3.txt was directly accessible.

    /license: Viewing this page didn't reveal much initially, but when I opened the browser inspector menu, I noticed an odd, encoded string with a trailing ==. I instantly recognized this as Base64!

Base64 Decoding:
I copied the string and decoded it, revealing a set of login credentials:

Elliot:<PASSWORD_REFERENCE>

The password was an obvious reference to the Mr. Robot TV show, confirming I was on the right track.

3. Gaining a Reverse Shell via WordPress

With credentials in hand, I navigated to the discovered WordPress login page (/wp-login.php).

WordPress Dashboard Access and File Edit

The Elliot credentials worked perfectly, giving me access to the dashboard. My first thought was to look for file modification capabilities.

    I went to Appearance → Editor.

    I selected the 404 Template (404.php) file. Since this file executes server-side when a page isn't found, it was the perfect place to drop a shell.

Deploying the PHP Reverse Shell

    I grabbed a PHP reverse shell script and customized it with my attacking machine's IP and a chosen port (e.g., 9001).

    I replaced the entire contents of 404.php with my PHP shell code.

    I set up my listener in a separate terminal:
    Bash

    nc -lvnp 9001

    Triggering the Shell: To execute the code, I just visited the page in my browser:

    http://<TARGET_IP>/404.php

Result: Boom! I caught a low-privilege shell as the web server user. I then upgraded it for a stable terminal experience:
Bash

python -c 'import pty; pty.spawn("/bin/bash")'

4. Privilege Escalation to User robot

Now that I had a foothold, I began enumerating the filesystem for internal credentials, starting with the home directories. Inside /home/robot/, I found two files:

Discovery of Hashed Password and Key 2

    key-2-of-3.txt: Unreadable by my current user.

    password.raw-md5: This file held a password hash for the user robot.

Bash

cat /home/robot/password.raw-md5
# Output: robot:<MD5_HASH>

Hash Cracking

I identified the hash as MD5. Rather than jumping straight to a lengthy brute-force, I checked the hash against an online rainbow table (like CrackStation) for a rapid solution.

Result: The password cracked instantly! It was a very long, but simple, dictionary-based string (e.g., abcdefghijklmnopqrstuvwxyz).

Switching User

I used the new credentials to switch my user context:
Bash

su robot
# Paste password

Success! I was now the robot user. The second key was now readable:
Bash

cat /home/robot/key-2-of-3.txt

5. Final Privilege Escalation to Root (Key 3)

The last piece of the puzzle was getting root access for the final key.

SUID Binary Enumeration

I searched the entire system for files with the SUID (Set User ID) bit set, which is a common privilege escalation vector.
Bash

find / -perm /6000 2>/dev/null | grep bin

The output was long, but one entry jumped out as unusual: /usr/bin/nmap.

Exploiting Nmap SUID

I quickly confirmed on GTFOBins that nmap with the SUID bit can be abused. The trick is to enter its interactive mode and then spawn a shell.

    Execute Nmap:
    Bash

nmap --interactive

Spawn a shell (using the ! command):
Bash

    nmap> !sh

Result: Because Nmap was running with root privileges (due to the SUID bit), the shell it spawned was also a root shell!
Bash

whoami
# Output: root

Retrieving Key 3

With root access, the final step was trivial. I went straight to the root directory and grabbed the final key:
Bash

cat /root/key-3-of-3.txt

Conclusion

I successfully completed the entire TryHackMe Mr. Robot CTF. It was a rewarding process that tested multiple phases of penetration testing. I found all three keys and achieved root access!

