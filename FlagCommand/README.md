Flag Command - Hack The Box Writeup
Challenge Information
Platform: Hack The Box

Category: Web

Difficulty: Very Easy

Objective
The goal was to navigate a web-based "Terminal Game" and find a hidden command that provides the flag.

Walkthrough
1. Initial Exploration
The challenge presents a browser-based game that mimics a terminal interface. I played through the game normally at first, but died twice by choosing the standard available options. I realized the "intended" path was likely hidden from the visible menu.

2. Inspecting the Source
I opened the Browser Developer Tools (F12) to inspect the application's structure. I focused on the Sources tab to examine the JavaScript (JS) files driving the game logic.

<img width="1908" height="918" alt="Screenshot 2025-12-31 182234" src="https://github.com/user-attachments/assets/35d7c061-4749-4276-b3ab-39bf394ecece" />

Observations: I found several JS files. While the flag wasn't written directly in the code, I noticed the logic suggested the existence of a "secret" or "hidden" input that wasn't listed in the game's standard prompt.

<img width="1111" height="139" alt="Screenshot 2025-12-31 182354" src="https://github.com/user-attachments/assets/797738a6-5a11-4bcb-b2ce-84913ea6dc16" />

3. Monitoring Network Traffic & Console
I restarted the game while keeping the Network and Console tabs open to see how the game fetched its list of commands.

The Discovery: I noticed the game fetched a resource (often a JSON file or an endpoint like /api/options).


The Secret: Upon inspecting this "options" data in the console, I found a list of standard commands (e.g., HEAD NORTH, FOLLOW PATH) and a secret section containing a command not shown in the game UI.
<img width="596" height="628" alt="image" src="https://github.com/user-attachments/assets/26c14a13-ce70-48b3-b47e-c20a1fe2bbc6" />

4. Capturing the Flag
I returned to the game terminal and manually typed the hidden command I discovered in the console.

Result: The game recognized the command and printed the flag.
<img width="1325" height="264" alt="image" src="https://github.com/user-attachments/assets/4699a66d-e065-4867-a4cd-f742315763a5" />

Flag
HTB{w3b_1nsp3ct_4nd_c0ns0l3_f72q} (Replace this with the actual flag you found!)

Key Takeaways
Client-Side Secrets: Never trust the client. If a list of "allowed" commands is sent to the browser, a user can always find them by inspecting the traffic or the console.

API Inspection: Checking the "Network" tab for JSON responses is a fundamental skill for Web Pentesting.

Code Review: Reading through Javascript files can reveal hidden logic or "hidden features" left behind by developers.
