Hack The Box: Crocodile Walkthrough
Target IP: 10.129.x.x
Date: March 2026
Category: Web / Linux / Easy

1. Executive Summary
Crocodile is an "Easy" rated Linux machine that focuses on standard web enumeration and credential harvesting. The attack path involves identifying hidden directories via directory brute-forcing, retrieving sensitive files through an exposed service, and utilizing found credentials to access an administrative dashboard.

2. Reconnaissance & Enumeration
Phase 1: Port Scanning
I began with a standard Nmap scan to identify open ports and services:
nmap -sC -sV -oN nmap_scan.txt 10.129.x.x

Results:

Port 21 (FTP): vsftpd 3.0.3 (Anonymous login allowed)

Port 80 (HTTP): Apache httpd 2.4.41

Phase 2: FTP Enumeration
Since anonymous login was enabled, I accessed the FTP server to check for sensitive files:
ftp 10.129.x.x (Logged in as anonymous)

Files Found:

allowed.userlist

allowed.userlist.passwd

Note: These files contained a list of potential usernames and their corresponding passwords, which are critical for the next phase.

3. Web Enumeration
I navigated to the web server on port 80. The landing page was a static corporate site. To find administrative entry points, I performed directory brute-forcing.

Directory Brute-forcing
Using gobuster or ffuf:
gobuster dir -u http://10.129.x.x -w /usr/share/wordlists/dirb/common.txt -x php,html

Key Discovery:

/login.php (The administrative login portal)

4. Exploitation & Foothold
Administrative Access
Using the credentials harvested from the FTP service, I attempted to log in to /login.php.

Username: admin (or the specific name from your userlist)

Password: [The password you found]

Successful authentication granted access to the internal dashboard, where the user flag was located.

5. Post-Exploitation & Flags
User Flag
Once inside the dashboard, the user flag was displayed (or found in a specific file/directory).

Path: Dashboard / Home

<img width="1531" height="696" alt="image" src="https://github.com/user-attachments/assets/8a3e4ddc-0741-4ae0-bc00-f324174fa59e" />


6. Conclusion & Lessons Learned
Information Leakage: Anonymous FTP access provided the initial credentials. Never allow anonymous login on servers containing sensitive user lists.

Directory Brute-forcing: Standard wordlists are highly effective at finding hidden management interfaces like login.php.
