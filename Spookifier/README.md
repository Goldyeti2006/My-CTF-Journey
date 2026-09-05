# HTB Challenge: Spookifier - Writeup

## Challenge Information
* **Platform:** Hack The Box
* **Category:** Web
* **Difficulty:** Very Easy

---

## Objective
The goal of this challenge was to exploit a Server-Side Template Injection (SSTI) vulnerability in a Python-based web application to read the system's flag file.

---

## Investigation Steps

### 1. Initial Web Analysis
The application takes a user's name and "spookifies" it by displaying it in several different fonts. I first checked the **Developer Tools (Inspect)** tab, looking for hidden API keys or hardcoded flags in the source and console, but found no leads.

### 2. Source Code Review
I extracted the provided challenge files and analyzed the backend structure. 
* **Key Finding:** I identified that the app was built using **Python** and the **Flask** web framework.
* **Vulnerability Analysis:** By reviewing `app.py`, I noticed that the application used the **Mako** template engine to render the user's input directly into the page without proper sanitization.

### 3. Testing for SSTI
Based on the use of a template engine, I suspected **Server-Side Template Injection (SSTI)**. I tested this by entering a mathematical expression inside the Mako template syntax:

* **Payload:** `${7*7}`
* **Result:** The page rendered `49` instead of the literal string `${7*7}`.

This confirmed that the server was executing the code inside the brackets.



### 4. Exploitation (Remote Code Execution)
Since the template engine executes Python code, I used Python's built-in file handling functions to read the flag.

* **Payload used:** `${open('flag.txt').read()}`
* **Result:** The application executed the command and displayed the flag's contents on the webpage.

---

## Flag
`HTB{t3mpl4t3_1nj3ct10n_C4n_3x1st5_4nywh343!!}

---

## Key Takeaways
* **Source Code is Gold:** Identifying the language (Python) and the template engine (Mako) early allowed for targeted research.
* **SSTI Vulnerability:** Whenever user input is reflected back via a template engine, it must be sanitized. If not, an attacker can execute arbitrary Python commands.
* **Simplicity First:** Often, the most basic Python commands (like `open().read()`) are the most effective if the environment isn't heavily restricted.
