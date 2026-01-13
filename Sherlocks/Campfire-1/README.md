Project Title: HTB Sherlock - Campfire 1 (Kerberoasting Analysis)
1. Scenario & Objectives
Goal: Investigate a compromised host to identify a Kerberoasting attack, the tools used, and the timeline of events. Tools Used: Windows Event Viewer, PEcmd (Eric Zimmerman's Tools), Timeline Explorer (or VS Code).

2. Investigation Log
Phase 1: Initial Triage & The "Platform" Pivot
I initially attempted to parse the provided .evtx files in Kali Linux. However, due to the complexity of Windows Event Logs, I realized that using native Windows tools would be more efficient. I switched to a Windows environment to use Event Viewer and Zimmerman's tools.

Lesson: Always use the right tool for the OS. Parsing Windows artifacts is often smoother in a Windows environment.

Phase 2: Detecting the Attack (Event Logs)
I started by hunting for Kerberoasting indicators.

Event ID 4769 (Kerberos Service Ticket Requested): I searched for an unusual volume of ticket requests. I specifically looked for requests with Encryption Type 0x17 (RC4), which is a common indicator of Kerberoasting attacks since attackers downgrade encryption to crack it easier offline.

Findings:

Targeted Service: [Insert Service Name you found]

Workstation Name: Found within the network information of the event log.

Phase 3: PowerShell Forensics (Script Block Logging)
To understand how the attacker enumerated the network, I looked at Event ID 4104 (PowerShell Script Block Logging).

Discovery: I found the script code used to enumerate Active Directory objects.

The Timestamp Trap: I initially struggled to correlate the logs with the answer submission. My host machine was displaying logs in IST (UTC+5:30), but the challenge required UTC.

Fix: I converted the event timestamps to UTC to match the forensic timeline.

Phase 4: Execution Evidence (Prefetch Analysis)
I needed to prove that the attacker actually ran the Rubeus binary.

Tooling: I used PEcmd to parse the Prefetch files.

PowerShell

PEcmd.exe -d "Path\To\Prefetch" --csv . --csvf result.csv
Analysis: The tool generated two CSVs. The first was too verbose, but the second contained the execution timeline.

Finding Rubeus: I filtered for "Rubeus" and found the exact execution time.

Path Correction: The tool output the raw Volume Serial (VOLUME{...}), but I had to map this mentally to the Logical Drive (C:\) to confirm the full path: C:\Users\Alonzo.spire\Downloads\Rubeus.exe.

3. Key Challenges & Lessons Learned
Timezone Hygiene: One of the biggest hurdles was the time difference. I spent a considerable amount of time debugging "wrong" answers that were actually just "timezone shifted" answers. I now know to always standardize investigation timelines to UTC immediately.

Tool Output Filtering: PEcmd is powerful but noisy. Learning to ignore the "Timeline" CSV and focus on the "Files" CSV saved me time.

Volume GUID vs. Logical Paths: Forensic tools often give the raw disk ID. I learned that for reporting (and CTF flags), this usually needs to be translated back to the human-readable C:\ path.
