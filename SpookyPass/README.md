SpookyPass - Hack The Box Writeup
Challenge Information
Platform: Hack The Box

Category: Reversing

Difficulty: Very Easy

Objective
The goal of this challenge was to extract the flag from a provided Linux executable file.

Walkthrough
1. Initial Attempts & Troubleshooting
After downloading and extracting the challenge file spookypass.zip, I initially attempted to run the binary by double-clicking it and using "Open with Browser." Both methods failed because the file is a Linux-specific executable and does not have a graphical interface.

2. File Identification
I moved to the terminal to identify the file type. I used the file command to understand what I was dealing with:

Bash

file pass
Result: The output confirmed the file was an ELF 64-bit LSB executable. This told me it was a compiled program designed to run in a Linux environment.

3. Static Analysis (Finding the Password)
Instead of running the file immediately, I performed Static Analysis using the strings command. This tool searches for human-readable sequences of characters embedded within the binary code.

Bash

strings pass


Scanning through the output, I identified a unique string that looked like a password."s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5"

Note: At first, I mistakenly tried to submit this string as the flag, but I realized it was the "key" needed to unlock the actual flag.

4. Execution and Permission Handling
I tried to execute the file using ./pass, but it failed because the file did not have execution permissions. I corrected this using the chmod command:

Bash

# Granting execute permissions
chmod +x pass

# Running the program
./pass
5. Capturing the Flag
Once the program was running, it prompted me for a password. I entered the string found during the strings analysis. The program validated the password and printed the flag to the terminal.

FlagHTB{un0bfu5c4t3d_5tr1ng5}

Key Takeaways
File Permissions: Linux binaries require the +x bit to be set before they can be executed.

Strings Utility: Simple hardcoded credentials can often be found without ever running the program by using static analysis tools.

CLI over GUI: Security tools and binaries are best handled through the command line rather than a file explorer.
