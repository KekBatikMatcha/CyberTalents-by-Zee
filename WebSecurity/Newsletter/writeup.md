# Newsletter - CyberTalents

---

## Challenge Description
**The administrator put the backup file in the same root folder as the application, help us download this backup by retrieving the backup file name.**

## Challenge Link
https://cybertalents.com/challenges/web/newsletter

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="669" height="341" alt="image" src="https://github.com/user-attachments/assets/558a4f8e-182c-463a-8c3f-8172f5490183" />
</p>

---

## 2. Challenge Page — Newsletter Subscription Form

Upon navigating to the challenge URL, a simple webpage titled **"Super Newsletter"** is presented. It contains a single email input field and a submit button. The goal appears to be subscribing an email to a newsletter.

<p align="center">
<img width="476" height="239" alt="image" src="https://github.com/user-attachments/assets/828981ea-33ce-4015-b1ee-6e0dab48a32f" />
</p>

---

## 3. Source Code Analysis — Nothing Suspicious

Inspecting the page source reveals standard HTML with no obvious vulnerabilities, hidden fields, or exposed file paths. The attack surface here is the email input field itself.

<p align="center">
<img width="850" height="406" alt="image" src="https://github.com/user-attachments/assets/671cf46e-c061-4e9b-b8bd-5c7188ec9895" />
</p>

---

## 4. Testing Input Validation

Various inputs are tested to understand how the server validates the email field.

**Test 1 — Any random value is accepted:**

Submitting any random string as long as it is not a registered email returns a valid response — the server does not check if the email actually exists.

<p align="center">
<img width="386" height="218" alt="image" src="https://github.com/user-attachments/assets/e6a60040-fc12-4268-b7d5-bff535805d3e" />
</p>

**Test 2 — Must contain `@` and `.`:**

Submitting a value without `@` or `.` returns an error — the server enforces basic email format.

<p align="center">
<img width="331" height="58" alt="image" src="https://github.com/user-attachments/assets/51908321-37f5-4c31-a848-6dc31bfff8da" />
</p>

**Test 3 — Some special characters are blocked:**

Certain special characters trigger a validation error — the server has a basic filter blocking some inputs.

<p align="center">
<img width="347" height="139" alt="image" src="https://github.com/user-attachments/assets/88e1b9be-4203-4696-968e-b0cd5645e57e" />
</p>

**Summary of validation behaviour:**

| Input | Server Response |
|-------|----------------|
| Any string with `@` and `.` | Accepted — no real email verification |
| Missing `@` or `.` | Rejected — basic format enforced |
| Certain special characters | Rejected — basic filter present |

The filter is partial — it blocks some characters but not all. This suggests a **Command Injection** vulnerability may be possible if the right bypass characters are used.

---

## What is Command Injection?

**Command Injection** is a vulnerability that occurs when user-supplied input is passed to a system shell command on the server without proper sanitisation. If the server uses the email input directly in a shell command — for example to write the email to a file — an attacker can append additional shell commands to it.

### How Shell Command Separators Work

In Linux/Unix shell, multiple commands can be chained together using special separator characters:

| Separator | Syntax | Behaviour |
|-----------|--------|-----------|
| `;` | `cmd1; cmd2` | Run `cmd1` then always run `cmd2` regardless of result |
| `\|\|` | `cmd1 \|\| cmd2` | Run `cmd2` only if `cmd1` fails |
| `&&` | `cmd1 && cmd2` | Run `cmd2` only if `cmd1` succeeds |
| `\|` | `cmd1 \| cmd2` | Pipe output of `cmd1` into `cmd2` |

So if the server runs something like:

```bash
echo "user@email.com" >> emails.txt
```

And the attacker submits:

```
user@email.com; ls
```

The server executes:

```bash
echo "user@email.com; ls" >> emails.txt
```

Or in an unsanitised case:

```bash
echo "user@email.com"; ls
```

Which runs both `echo` and `ls` — listing the server's directory contents.

---

## 5. Intercepting the Request — Command Injection via Email Field

The form submission is intercepted using **Burp Suite**. The email parameter in the POST request body is modified to include a command injection payload.

The semicolon `;` is appended after a valid email address, followed by the command `ls || la`:

```
email=test@gmail.com; ls || la
```

<p align="center">
<img width="709" height="362" alt="image" src="https://github.com/user-attachments/assets/b771625b-622f-4225-89f0-bc9065f13450" />
</p>

### Why Semicolon `;` After the Email?

The `;` character is a **shell command separator** in Linux. By placing it after the email address, we close the first command (whatever the server does with the email) and start a new separate command. This means:

```
test@gmail.com; ls || la
```

Gets interpreted by the shell as two separate commands:
1. The original email handling command with `test@gmail.com`
2. Then `ls || la` as a completely new command

### Why `ls || la`?

`ls` is the standard Linux command to list files in the current directory. `|| la` is added as a fallback — `||` means "if the first command fails, run the second." `la` is not a real command but it ensures output is still returned if `ls` behaves differently.

The `||` operator is used here as a bypass technique — some filters block single `|` but not `||`.

In short: `ls || la` means **"list the directory files"** — which reveals what files exist in the server's current working directory.

---

## 6. Server Response — Directory Contents Revealed

The server executes the injected command and returns the directory listing in the response:

```
test3@gmail.com
emails_secret_1337.txt
hgdr64.backup.tar.gz
index.php
```

<p align="center">
<img width="164" height="94" alt="image" src="https://github.com/user-attachments/assets/3c762c77-5f5d-48e6-8b14-6ebee6b76d4f" />
</p>

The directory contains:

| File | Description |
|------|-------------|
| `test3@gmail.com` | A stored email from a previous submission |
| `emails_secret_1337.txt` | A file storing subscribed emails |
| `hgdr64.backup.tar.gz` | **The backup file the challenge asks us to find** |
| `index.php` | The main application file |

The backup file name is: **`hgdr64.backup.tar.gz`**

---

## 7. Obtaining the Flag

Read Back the descripton: 

The challenge description asks to **retrieve the backup file name** — so the flag is the filename itself:

> **FLAG: `hgdr64.backup.tar.gz`**

---

# OWASP Classification

This challenge demonstrates two vulnerabilities:

> **OWASP Top 10: A03 — Injection (Command Injection)**
> The email input field was passed directly into a server-side shell command without proper sanitisation. Appending `;` followed by a shell command allowed arbitrary command execution on the server.

> **OWASP Top 10: A05 — Security Misconfiguration**
> The backup file was stored inside the web root directory — making it directly downloadable via URL once its filename was known. Backup files should never be stored in publicly accessible directories.

---

# Why This Is a Security Issue

The application had two independent security failures that together allowed the backup file to be discovered and downloaded:

| Vulnerability | Impact |
|--------------|--------|
| Email input passed to shell without sanitisation | Attacker can run any Linux command on the server |
| `ls` command reveals all files in the directory | Backup filename discovered instantly |
| Backup file stored inside web root | Directly downloadable via URL — no authentication required |
| Partial input filter | Only some characters blocked — `;` and `\|\|` bypassed the filter |

### Why Partial Filters Fail

The application blocked some special characters but not `;` and `||`. This is a classic example of a **blocklist approach** — trying to block known bad characters rather than only allowing known safe ones. Blocklists always fail because:

- There are too many possible bypass techniques to anticipate
- Different character encodings can bypass string matching filters
- Shell operators vary across environments

The correct fix is to **never pass user input to shell commands** — use language-native functions instead (e.g. PHP's `file_put_contents()` instead of `echo >> file`).

---

# Exploitation Flow (OWASP A03 + A05 Mapping)

1. Attacker accesses the Newsletter subscription page
2. Source code inspection reveals nothing obvious
3. Input validation testing shows `@` and `.` are required — some characters blocked
4. Burp Suite intercepts the POST request
5. `;` appended after valid email — shell command separator
6. `ls || la` appended — Linux directory listing command
7. Server executes the injected command and returns directory listing
8. Four files revealed — `hgdr64.backup.tar.gz` identified as the backup file
9. Backup filename appended to the URL — file downloads successfully
10. Flag retrieved: **`hgdr64.backup.tar.gz`**

---

# Conclusion

This challenge demonstrates **Command Injection** — one of the most critical web vulnerabilities. The server passed the email input directly into a shell command, allowing an attacker to append arbitrary commands using the `;` separator. A partial character filter provided a false sense of security — it was trivially bypassed using `||` instead of `|`.

The backup file being stored in the web root compounded the issue — once the filename was known via command injection, the file was immediately downloadable without any authentication.

To prevent command injection, applications should:

- **Never pass user input to shell commands** — use language-native file operations instead
- Use an **allowlist approach** for input validation — only permit known safe characters
- **Store backup files outside the web root** — never in a publicly accessible directory
- **Rename backup files** with unpredictable names — but this is defence in depth, not a primary fix
- Apply the **principle of least privilege** — the web server process should not have permission to list or read sensitive files
