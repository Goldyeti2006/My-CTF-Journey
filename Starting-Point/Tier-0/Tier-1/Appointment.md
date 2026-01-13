Project Title: HTB Starting Point - Appointment (SQL Injection)
1. The Setup: "Leveling Up" to Local VPN
Objective: Establish a stable, persistent connection to the target without relying on the browser-based Pwnbox. Motivation: Due to free-tier restrictions on Pwnbox, I was forced to configure my local environment. While initially intimidating, setting up OpenVPN on Kali Linux proved to be straightforward and offered a much smoother experience.

Process:

Downloaded the .ovpn configuration file from HTB.

Connected via terminal: sudo openvpn starting_point.ovpn.

Verified the tun0 interface was active using ip a.

<img width="1240" height="718" alt="Screenshot from 2026-01-13 06-47-38" src="https://github.com/user-attachments/assets/17b62146-9d22-4531-a847-5b6857b1b9e1" />

Key Takeaway: The local setup is superior for latency and tool management. I now have a dedicated "attack box" environment ready to go.

2. Reconnaissance: Enumeration Strategy
I adopted a two-step scanning methodology to balance speed and detail.

Step 1: Initial Scan A quick sweep to identify open ports.

Result: Port 80 (HTTP) was open.

Step 2: Targeted Service Scan Once I knew the web server was the target, I ran a more aggressive scan to footprint the service.

Bash

nmap -sC -sV -p80 <TARGET_IP>
-sC: Ran default scripts (checking for common vulnerabilities).

-sV: Detected the specific version of the web server.
<img width="1240" height="380" alt="Screenshot from 2026-01-13 06-48-01" src="https://github.com/user-attachments/assets/13d225ad-ec5a-49b2-8986-929e8772aa91" />

3. Troubleshooting: The "Format" Trap (Task 4)
Challenge: Identifying the specific Apache version running on the target. The Issue: My Nmap scan correctly identified the server header as Apache httpd 2.4.38 ((Debian)). However, the platform rejected my initial answers due to strict formatting requirements. Solution: I realized the platform often filters out "noise" like OS identifiers (Debian) or service names (httpd).

Correct Answer: 2.4.38

Lesson: In CTFs and reporting, identifying the core data point is often as important as finding the raw data.

4. Exploitation: Authentication Bypass (SQL Injection)
Objective: Access the administrative dashboard without credentials. Vulnerability: The login form was susceptible to SQL Injection (SQLi), likely due to unsanitized user input in the backend query.

The Payload:

SQL

admin' #
Why it worked:

admin: The target username.

': Closes the data field in the SQL query.

#: Comments out the rest of the query (ignoring the password check entirely).

This manipulated the backend query to look something like SELECT * FROM users WHERE username = 'admin' # ..., effectively logging me in as the administrator instantly.
<img width="1140" height="259" alt="Screenshot from 2026-01-13 06-47-15" src="https://github.com/user-attachments/assets/b81d638a-a7d9-4141-a30c-93aa552c4882" />

5. Conclusion
This box was a confidence booster. I successfully migrated to a professional local setup and solved network enumeration and web exploitation challenges independently. The "intimidation factor" of using the command line for VPNs is gone.
Tools: Nmap, OpenVPN, Kali Linux (Local VM)
