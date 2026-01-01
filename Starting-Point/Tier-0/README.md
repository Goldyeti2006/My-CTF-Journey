HTB Starting Point - Tier 0 Summary
Overview
Tier 0 focuses on the absolute fundamentals of network service exploitation: Enumeration and Misconfigurations. These machines demonstrate how open ports and default settings can lead to full system compromise.

🐱 Machine 1: Meow
Key Concept: Telnet & Default Credentials.

Process: I used nmap to find that Port 23 (Telnet) was open. I learned that Telnet is unencrypted and often has a "root" user with no password.

Takeaway: Always disable legacy protocols like Telnet in favor of SSH.

🦌 Machine 2: Fawn
Key Concept: FTP (File Transfer Protocol).

Process: Nmap revealed Port 21 (FTP). I tested for Anonymous Login, which allowed me to log in without a password and download the flag.txt directly.

Takeaway: Anonymous access to file servers is a critical security risk.

💃 Machine 3: Dancing
Key Concept: SMB (Server Message Block) Enumeration.

Process: I used smbclient to list shares. I discovered a "Workshares" directory that was accessible without a password. I navigated the directories via the CLI to find the flag.

Takeaway: Network shares must be properly partitioned and password-protected.

🍷 Machine 4: Redeemer (or Explosion/Synced)
Key Concept: NoSQL/Redis Database (or RDP for Explosion).

Process: * If Redeemer: I used redis-cli to connect to Port 6379 and used the INFO and KEYS * commands to dump the database content.

If Explosion: I used xfreerdp to connect to a Windows machine via RDP using a blank password for the Administrator.

Takeaway: Databases and Remote Desktops are high-value targets; never leave them exposed to the public internet without authentication.

Technical Skills Demonstrated
Scanning: Using nmap -sV to identify service versions.

Protocol Interaction: Manually interacting with FTP, Telnet, and SMB clients.

Information Gathering: Identifying "low-hanging fruit" like default passwords and anonymous access.
