# WingData HTB: From Unauthenticated RCE to CVE-2025-4517 Root

# 📝 Executive Summary
WingData is a easy-to-medium Linux machine on Hack The Box that highlights the dangers of misconfigured third-party applications and the critical nature of path traversal vulnerabilities in standard libraries. This assessment covers initial access via Wing FTP Server, lateral movement through credential cracking with custom salts, and a final privilege escalation leveraging a logic flaw in the Python `tarfile` module (**CVE-2025-4517**).

---

# 🛠️ Phase I: Initial Access & Latency Management
The target was found running **Wing FTP Server**. An unauthenticated Remote Code Execution (RCE) vulnerability was exploited via the Web Admin interface using **Burp Suite**.

### **The "Hacker Latency" Challenge**
Due to the use of a mobile hotspot and VPN routing, the initial reverse shell was extremely laggy. 
* **The Fix:** Immediate TTY stabilization was required to prevent the shell from dropping during command execution.
  ```bash
  python3 -c 'import pty; pty.spawn("/bin/bash")'
  # Ctrl+Z
  stty raw -echo; fg
  export TERM=xterm

 # 🔑 Phase II: Lateral Movement (wacky)
After gaining a shell as the wingftp service account, an audit of the application directory was performed.

Credential Harvesting
Configuration files were located in /opt/wftpserver/Data/. A grep search for "password" revealed a SHA-256 hash for the user wacky.

The Salt: The server used a static salt: WingFTP.

Cracking: Using the salt and the hash 32940defd3c3ef70..., the plaintext password was recovered: !#7Blushing^*Bride5.

Access: Established a stable SSH session: ssh wacky@10.129.244.106.

# 🚀 Phase III: Privilege Escalation (CVE-2025-4517)
Auditing sudo privileges revealed a vulnerable Python script:
NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *

The Vulnerability: Link-Type Confusion
The script used tarfile.extractall(filter="data"). Despite the "data" filter's intent to block path traversal, it contains a logic flaw regarding Hardlinks pointing through Symlinks.  

Exploit Steps
Directory Nesting: Created a 16-level deep directory structure with 247-character names to confuse path resolution.

The Escape: Created a symbolic link named escape pointing to /etc.

The Bypass: Created a hardlink pointing through the escape symlink to the sudoers file.

The Trigger: Since the target machine lacked internet access, the exploit was transferred via scp and executed:
### Bash
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_999.tar -r restore_pwn  
Outcome: Successfully appended a NOPASSWD entry for wacky into /etc/sudoers.

# 🚩 Conclusion
User Flag: 32940defd3c3ef70a2dd44a5301ff984...  
Root Flag: 8783994f8a503ca...

# 🛡️ Remediation Recommendations
Update Python: Ensure Python is running version 3.12.11+ or 3.13.4+ to patch the tarfile logic.  

Restrict Sudo: Never allow wildcards (*) in sudoers entries for scripts that handle file archives.  

Manual Filtering: Implement custom Python logic to explicitly ban SYMTYPE and LNKTYPE members within tarfile.getmembers().  
