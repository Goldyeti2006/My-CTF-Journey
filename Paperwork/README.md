# Hack The Box — Paperwork

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Paperwork-green)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)

> **Platform:** Hack The Box  
> **Machine:** Paperwork  
> **OS:** Linux  
> **Difficulty:** Easy

---

# 1. Overview

Paperwork is a Linux machine that demonstrates exploitation of custom
network services, printer protocols, Linux IPC mechanisms, and
privilege escalation through a root-level management daemon.

The overall attack chain was:

```text
                    ┌─────────────────────┐
                    │   Network Scan      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  TCP/1515 LPD       │
                    │  Custom Service     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Command Injection   │
                    └──────────┬──────────┘
                               │
                               ▼
                             lp
                               │
                               ▼
                    ┌─────────────────────┐
                    │ TCP/9100            │
                    │ JetDirect / PJL     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Filesystem Abuse    │
                    │ / Path Traversal    │
                    └──────────┬──────────┘
                               │
                               ▼
                    archivist SSH Access
                               │
                               ▼
                    ┌─────────────────────┐
                    │ LinPEAS Enumeration │
                    └──────────┬──────────┘
                               │
                               ▼
                 /var/run/paperwork/mgmt.sock
                               │
                               ▼
                    Root Management Daemon
                               │
                               ▼
                         SCM_RIGHTS
                               │
                               ▼
                Privileged File Descriptor
                               │
                               ▼
                 Administrator Configuration
                               │
                               ▼
                             ROOT
```
2. Enumeration
2.1 Initial Nmap Scan

The first step was to identify the exposed attack surface.
```
nmap -sC -sV <TARGET_IP>
```

For a complete port scan:
```
nmap -p- --min-rate 5000 <TARGET_IP>
```

Observations

The scan revealed several interesting services, including:

SSH
A custom LPD-related service on TCP/1515
A printing service on TCP/9100

The custom printing infrastructure became the primary attack surface.

3. TCP/1515 — LPD Service

The service exposed on TCP/1515 did not behave like a normal HTTP
application.

Manual interaction was used to understand the protocol and determine
how commands and print-job information were processed.

Example:
```
nc <TARGET_IP> 1515
```
Screenshot

The captured protocol interaction showed the request/response behaviour
of the custom LPD service.

Request

Response

4. Command Injection

Further investigation of the LPD protocol showed that attacker-controlled
input could reach a command executed by the service.

This provided command execution on the target.

The important lesson was that custom protocols should not be treated as
"safe" simply because they are not HTTP-based.

Once command execution was obtained, the execution context was checked
with:
```
whoami
id
```
The initial foothold was obtained as:
```
lp
```
<img width="630" height="283" alt="Screenshot 2026-09-02 000536" src="https://github.com/user-attachments/assets/ca0dc25c-88bb-4e95-8e3a-15e25160fb14" />

Evidence note: Not every intermediate command was captured as a
screenshot during the original session. The screenshots included in
this write-up represent the evidence that was preserved during the
machine solve.

5. Enumerating Local Services

After obtaining the initial foothold, local services were enumerated.

A particularly interesting service was the printer service listening on
TCP/9100.
```
ss -lntup
```
The service was then investigated locally.

6. TCP/9100 — JetDirect

A scan of TCP/9100 was performed to identify the service.
```
nmap -sV -p 9100 <TARGET_IP>
```
The port was identified as a JetDirect/raw printing service.

Screenshot

The discovery of a printer service was important because printer
protocols such as PJL can expose functionality beyond simply submitting
print jobs.

7. PJL Investigation

The printer service was investigated using PJL commands.

The service exposed functionality that allowed interaction with the
underlying printer filesystem.

This ultimately provided a route to interact with files on the target
system.

The important concept here was:
```
Remote foothold
      |
      ▼
Local printer service
      |
      ▼
PJL filesystem functionality
      |
      ▼
File manipulation
```
8. SSH Key Abuse

The printer functionality was used to interact with the SSH configuration
of the archivist user.

The relevant file was:

/home/archivist/.ssh/authorized_keys

An attacker-controlled public key could then be used to authenticate as
the archivist user.

SSH access:
```
ssh archivist@<TARGET_IP>
```
The access level was verified with:
```
whoami
id
```
Expected context:

archivist
9. Post-Exploitation Enumeration

Once access as archivist was obtained, local privilege-escalation
enumeration was performed.

One of the primary enumeration tools used was LinPEAS.

LinPEAS

LinPEAS was transferred to the target and executed to identify:

SUID binaries
writable files
services
sockets
kernel information
scheduled tasks
credentials
possible kernel exploits
privilege-escalation vectors
Screenshot

10. Kernel Exploit Candidates

LinPEAS identified several potential kernel privilege-escalation
candidates.

The scan highlighted:

DirtyClone — CVE-2026-43503
pedit COW — CVE-2026-46331
GhostLock — CVE-2026-43499
Screenshot

These findings were treated as potential attack paths, rather than
automatically assuming that one of the kernel vulnerabilities was the
intended route.

This is an important penetration-testing lesson:

Enumeration tools identify possible vulnerabilities. They do not prove
that a vulnerability is exploitable or that it is the intended attack
path.

The machine contained a more interesting application-specific privilege
escalation path.

11. Paperwork Management Socket

Further local enumeration revealed a Unix domain socket:

/var/run/paperwork/mgmt.sock

Unix domain sockets should be treated as local services and investigated
in the same way as network services.

Useful enumeration:

find /var/run -type s 2>/dev/null

and:

ss -lx

The management socket belonged to a root-level Paperwork management
service.

12. Root Management Daemon

The management daemon was investigated to understand how it communicated
with clients through the Unix socket.

The important privilege boundary was:

archivist
   |
   | Unix domain socket
   ▼
root management daemon

Because the server process was running with root privileges, functionality
exposed through the socket became a potential privilege-escalation
surface.

13. SCM_RIGHTS

The privilege escalation involved Unix file descriptor passing using
SCM_RIGHTS.

SCM_RIGHTS is a Unix socket mechanism that allows one process to pass an
already-open file descriptor to another process.

Conceptually:
```
┌──────────────────────┐
│ archivist process    │
└──────────┬───────────┘
           │
           │ Unix socket
           │ SCM_RIGHTS
           ▼
┌──────────────────────┐
│ Root management      │
│ daemon               │
└──────────┬───────────┘
           │
           ▼
     privileged FD
           │
           ▼
      protected file
```
This was the key concept behind the final privilege escalation.

14. Administrator Configuration

The privileged file-access functionality allowed access to the Paperwork
administrator configuration.

The relevant configuration file was:

/etc/paperwork/admin_pins.conf

The configuration contained administrator authentication material.

For security reasons, credentials are intentionally redacted from this
write-up.

Example:
```
ADMIN_PASSWORD=[REDACTED]
```
Never commit the actual credential to a public GitHub repository.

15. Root Access

The recovered administrator credential was used to obtain elevated
access.

Privilege level was verified with:
```
whoami
id
```
Expected result:

root
uid=0(root)

16. Attack Chain Summary

The complete attack path was:
```
Nmap
  │
  ▼
TCP/1515<img width="630" height="283" alt="Screenshot 2026-09-02 000536" src="https://github.com/user-attachments/assets/522b954f-c477-499d-9a1b-e9f60ee782dd" />

  │
  ▼
Custom LPD Protocol
  │
  ▼
Command Injection
  │
  ▼
lp
  │
  ▼
TCP/9100
  │
  ▼
JetDirect / PJL
  │
  ▼
Filesystem Manipulation
  │
  ▼
SSH authorized_keys
  │
  ▼
archivist
  │
  ▼
LinPEAS
  │
  ▼
/var/run/paperwork/mgmt.sock
  │
  ▼
Root Management Daemon
  │
  ▼
SCM_RIGHTS
  │
  ▼
Privileged File Access
  │
  ▼
/etc/paperwork/admin_pins.conf
  │
  ▼
Administrator Credential
  │
  ▼
ROOT
```
17. Key Lessons Learned
Custom Services

Custom network services can be more interesting than standard services.

Always investigate:
```
nmap -sC -sV <TARGET>
```
and manually interact with unusual ports.

Localhost Services

After obtaining a foothold, always enumerate local services.
```
ss -lntup
```
A service that is not externally accessible may become exploitable after
obtaining local code execution.

Printer Protocols

JetDirect and PJL are not limited to printing.

Printer management protocols can expose filesystem and administrative
functionality.

Unix Domain Sockets

Unix sockets are local IPC endpoints and should be included in privilege
escalation enumeration.

find /var/run -type s 2>/dev/null
Linux IPC

Understanding mechanisms such as:

SCM_RIGHTS

can reveal privilege-escalation opportunities that automated tools may
not immediately explain.

Don't Blindly Trust LinPEAS

LinPEAS is excellent for enumeration, but a finding such as:

DirtyClone
pedit COW
GhostLock

does not automatically mean:

vulnerability = exploitable = intended path

Manual investigation is still required.

18. Useful Commands
```
Nmap
nmap -sC -sV <TARGET_IP>
nmap -p- --min-rate 5000 <TARGET_IP>
nmap -sV -p 9100 <TARGET_IP>
Service Interaction
nc <TARGET_IP> 1515
nc 127.0.0.1 9100
Local Enumeration
id
whoami
sudo -l
ss -lntup
ss -lx
find /var/run -type s 2>/dev/null
SSH
ssh archivist@<TARGET_IP>
LinPEAS
./linpeas.sh
```
19. Evidence

The following evidence was preserved from the original machine session:

Nmap enumeration
LPD server interaction
LPD request/response traffic
TCP/9100 enumeration
JetDirect identification
LinPEAS enumeration
LinPEAS privilege-escalation findings

Not every intermediate step was captured as a screenshot during the
original solve.

Screenshots included in this repository represent the evidence that was
actually preserved.

20. Disclaimer

This write-up was created for educational purposes as part of an
authorized Hack The Box machine.

All exploitation was performed against the designated Hack The Box
environment.

No real-world systems were targeted.
