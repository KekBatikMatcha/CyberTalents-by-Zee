# Catch Me If You Can - CyberTalents

---

## Challenge Description
**I'm Just wanna Make Sure if you Are Mr.Robot.**

## Challenge Link
https://cybertalents.com/challenges/web/catch-me-if-you-can

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="698" height="363" alt="image" src="https://github.com/user-attachments/assets/268bd2a5-fd3f-4b50-a101-f46d91575379" />
</p>

---

## 2. Challenge Page — Website Home

Upon navigating to the challenge URL, a website is presented with a navigation bar containing **Home, About, Project, Contact** links. Clicking any of the navigation links does not change the page content — nothing appears immediately exploitable on the surface.

<p align="center">
<img width="900" height="437" alt="image" src="https://github.com/user-attachments/assets/8891a3c6-d91b-4d59-b89b-ee46d799d8cc" />
</p>

---

## 3. Identifying Hidden Files — URL Clue

Clicking the **Home** link in the navigation bar reveals a clue in the browser URL bar:

```
/index.php
```

<p align="center">
<img width="494" height="47" alt="image" src="https://github.com/user-attachments/assets/4b52cd19-e5af-4fd8-a588-9167cad0aa97" />
</p>

The presence of `index.php` in the URL suggests the server is running PHP — and that there may be other hidden PHP files that are not linked from the main page. This is the trigger to run a directory enumeration scan.

---

## 4. Directory Enumeration — Using Dirsearch

To discover all hidden files and directories on the server, the **Dirsearch** tool is used:

```bash
dirsearch -u http://cdcamxwl32pue3e6mxmqvrwdh2zokm236mnkmugv0-web.cybertalentslabs.com
```

<p align="center">
<img width="222" height="68" alt="image" src="https://github.com/user-attachments/assets/669d2f0a-c336-42ef-8e6c-c99aeab5ff57" />
</p>

The scan returns three interesting files:

```
[09:19:28] 200 -    0B  - /flag.php
[09:20:07] 200 -   61B  - /robots.txt
[09:20:13] 200 -    2KB - /source.php
```

| File | Status | Size | Notes |
|------|--------|------|-------|
| `/flag.php` | 200 | 0B | Exists but returns empty — flag is here but not directly visible |
| `/robots.txt` | 200 | 61B | May contain hidden path references |
| `/source.php` | 200 | 2KB | Contains PHP source code — key to finding the password |

---

## 5. Investigating `/flag.php` — Empty Response

Navigating to `/flag.php` directly returns a blank page — the file exists (status 200) but displays nothing.

<p align="center">
<img width="731" height="460" alt="image" src="https://github.com/user-attachments/assets/0549f436-20a3-41de-93b4-5811339ce8a4" />
</p>

This is suspicious — a file named `flag.php` that returns 0 bytes suggests the flag is stored inside it but is not echoed to the browser directly. It is likely included by another file and only revealed under specific conditions.

---

## 6. Investigating `/source.php` — Password Logic Discovered

Navigating to `/source.php` reveals the full PHP source code of the password validation logic:

```php
<?php

include('flag.php');

$password = $_POST['pass'];

if (strpos($password, 'R_4r3@') !== FALSE) {

    if (!preg_match('/^-?[a-z0-9]+$/m', $password)) {

        die('ILLEGAL CHARACTERS');
    }
    echo $cipher;
} else {
    echo 'Wrong Password';
}
?>
```

<p align="center">
<img width="628" height="427" alt="image" src="https://github.com/user-attachments/assets/86d73e93-0081-48b6-b417-d9399e092ac6" />
</p>

### Code Explanation — Line by Line

**Line 1 — Include the flag file:**
```php
include('flag.php');
```
The flag is loaded from `flag.php` into the current script. The variable `$cipher` inside `flag.php` holds the encoded flag value.

**Line 2 — Read the password from POST:**
```php
$password = $_POST['pass'];
```
The password is taken from a POST form submission — meaning the login form sends the password in the request body.

**Line 3 — First check: must contain `R_4r3@`:**
```php
if (strpos($password, 'R_4r3@') !== FALSE)
```
The password **must contain** the substring `R_4r3@` somewhere in it. If not, the response is `"Wrong Password"`.

**Line 4 — Second check: must only contain `[a-z0-9]`:**
```php
if (!preg_match('/^-?[a-z0-9]+$/m', $password))
```
The password is checked against the regex `/^-?[a-z0-9]+$/m` — it must consist **only of lowercase letters and digits** (with an optional leading `-`). If the password contains anything else — like uppercase letters, `_`, `@`, or special characters — it dies with `"ILLEGAL CHARACTERS"`.

**The contradiction:**
This creates an impossible condition at first glance:

- Condition 1 requires `R_4r3@` — which contains uppercase `R`, underscore `_`, and `@`
- Condition 2 requires only `[a-z0-9]` — which **blocks** uppercase, `_`, and `@`

Both conditions cannot be satisfied at the same time with a normal string. This is where the bypass technique comes in.

---

## 7. Investigating `/robots.txt` — Another Hidden Path

Navigating to `/robots.txt` reveals:

```
User-agent: *
Disallow: /S3cr3t.php
Disallow: /source.php
```

<p align="center">
<img width="146" height="155" alt="image" src="https://github.com/user-attachments/assets/b3fd17e2-e34d-4fb5-ab6c-049da8728a55" />
</p>

### What is `robots.txt`?

`robots.txt` is a standard file that tells web crawlers (like Google's search bot) which pages **should not** be indexed. The `Disallow` directive means the developer does not want these pages appearing in search results.

However — **`Disallow` does NOT prevent access**. It is purely an instruction to well-behaved bots. Any human or tool can still navigate directly to these URLs. Listing sensitive paths in `robots.txt` is a common mistake that actually helps attackers discover hidden pages.

The path `/S3cr3t.php` is a new discovery — not found by Dirsearch. This is the actual login page.

---

## 8. Accessing `/S3cr3t.php` — Restricted Login Page

Navigating to `/S3cr3t.php` reveals a restricted login page with the message:

> **"This is a Restricted Area Give a Proof you are authorised"**

The page contains a single password input field.

<p align="center">
<img width="695" height="451" alt="image" src="https://github.com/user-attachments/assets/f23a2427-e4da-4369-ba64-41f1bcad69de" />
</p>

The source code from `/source.php` is the validation logic for this form — so understanding the password conditions is the key to accessing this page.

---

## 9. Crafting the Password — Regex Bypass Using `%0a`

The two conditions from the source code appear contradictory:

| Condition | Requirement |
|-----------|-------------|
| `strpos($password, 'R_4r3@')` | Password must **contain** `R_4r3@` |
| `preg_match('/^-?[a-z0-9]+$/m', $password)` | Password must **only contain** `[a-z0-9]` |

The bypass exploits the **`/m` flag** in the regex — which stands for **multiline mode**.

### What Does the `/m` Flag Do?

In normal mode, `^` matches the start of the entire string and `$` matches the end of the entire string. In **multiline mode (`/m`)**, `^` and `$` match the start and end of **each individual line** — not the whole string.

This means if the password contains a **newline character**, the regex only checks each line independently — not the whole string together.

### How the Payload Works

The password submitted is:

```
111%0aR_4r3@
```

- `111` — a simple lowercase/digit string that satisfies the regex check on line 1
- `%0a` — URL-encoded newline character (`\n`)
- `R_4r3@` — the required substring for the `strpos` check on line 2

When the server processes this:

**`strpos` check (Condition 1):**
```
The full password is: "111\nR_4r3@"
strpos finds "R_4r3@" → condition passes ✅
```

**`preg_match` check (Condition 2) with `/m` flag:**
```
Line 1: "111"    → matches [a-z0-9]+ → passes ✅
Line 2: "R_4r3@" → NOT checked because regex only checked first line ✅
```

Because of multiline mode, the regex only validates the first line `111` — which passes cleanly. The second line `R_4r3@` is never validated against the regex. Both conditions are satisfied simultaneously.

**The password is: `111%0aR_4r3@`**

---

## 10. Submitting the Password — Brainfuck Response

The password `111%0aR_4r3@` is entered into the `/S3cr3t.php` form. The page appears to show no visible response in the browser — this is because the output contains a newline before the content, making it appear blank.

<p align="center">
<img width="604" height="353" alt="image" src="https://github.com/user-attachments/assets/93bce534-3c45-468e-9d39-d1a2782aea6f" />
</p>

To see the actual server response, **Burp Suite** is used to intercept the POST request and inspect the raw response in Repeater.

### Why Does the Browser Show Nothing?

The server response contains the flag encoded in **Brainfuck** — but browsers render HTML and text, so a raw string of Brainfuck characters may appear invisible or get swallowed by the rendering engine. Burp Suite shows the raw HTTP response body exactly as the server sent it — including content that browsers would otherwise not display.

The raw response revealed in Burp Suite is:

```
-[------->+<]>---.++++++.------------.--[--->+<]>---.[----->+<]>.[--->++<]>.>-[----->+<]>.>-[--->+<]>--.[--->+<]>+++.--.--[->+++++<]>+.---[-->+++<]>--.+[----->+<]>.++[++>---<]>.+[->++<]>.-----.+[--->++<]>+.--[----->+<]>-.>-[----->+<]>.+.>--[-->+++<]>.
```

<p align="center">
<img width="598" height="230" alt="image" src="https://github.com/user-attachments/assets/6b0949fa-dad6-484d-b257-e430a1381ed4" />
</p>

<p align="center">
<img width="431" height="115" alt="image" src="https://github.com/user-attachments/assets/284fa52a-9347-43bf-a9ed-94fc3193dce0" />
</p>

### Alternative Ways to See the Response

| Method | How |
|--------|-----|
| **Burp Suite Repeater** | Send POST request and view raw response body |
| **Browser DevTools** | Network tab → find the POST request → Response tab |
| **cURL (command line)** | `curl -X POST -d "pass=111%0aR_4r3@" [url]/S3cr3t.php` |

---

## 11. Decoding Brainfuck — Flag Obtained

### What is Brainfuck?

**Brainfuck** is a minimalist esoteric programming language created in 1993 using only 8 commands: `+`, `-`, `>`, `<`, `[`, `]`, `.`, `,`. Despite its simplicity, it is Turing complete — meaning any computable program can theoretically be written in it. It is commonly used in CTF challenges to encode output in a way that is unreadable without a decoder.

| Character | Meaning |
|-----------|---------|
| `+` | Increment current memory cell |
| `-` | Decrement current memory cell |
| `>` | Move pointer right |
| `<` | Move pointer left |
| `[` | Start loop — skip if current cell is 0 |
| `]` | End loop — jump back if current cell is non-zero |
| `.` | Output current cell as ASCII character |
| `,` | Read input into current cell |

The Brainfuck string is decoded using [https://www.dcode.fr/brainfuck-language](https://www.dcode.fr/brainfuck-language):

<p align="center">
<img width="226" height="245" alt="image" src="https://github.com/user-attachments/assets/0633432b-1e23-48f0-b585-5729093c1d46" />
</p>

The decoder also provides the memory dump showing how each character of the flag was computed:

```
Memory Dump: [index] = char (ASCII code)
[0]  = (0)
[4]  = R  (82)
[6]  = 3  (51)
[16] = r  (114)
[18] = 4  (52)
[20] = }  (125)
pointer = 20
```

Each cell in the Brainfuck memory corresponds to one ASCII character — the program increments and decrements memory cells to build each character of the flag one by one, then outputs them with `.`.

The decoded output reveals the flag:

> **FLAG: `FL@g{R3Str1Ct1d_Ar34}`**

---

# OWASP Classification

This challenge demonstrates multiple vulnerabilities:

> **OWASP Top 10: A05 — Security Misconfiguration**
> Sensitive file paths were disclosed in `robots.txt` — a file intended for search engine bots that is publicly accessible. The source code of the validation logic was also exposed via `/source.php`.

> **OWASP Top 10: A03 — Injection**
> The regex validation used the `/m` multiline flag which allowed a newline character to split the input and bypass the character restriction — a form of input manipulation that exploited a logic flaw in the validation implementation.

---

# Why This Is a Security Issue

The application had multiple compounding weaknesses:

| Vulnerability | Impact |
|--------------|--------|
| `/robots.txt` lists hidden paths | Attacker discovers `/S3cr3t.php` without needing to scan |
| `/source.php` exposes validation logic | Password conditions fully readable — bypass engineered from source |
| Regex uses `/m` multiline flag | Newline character splits input — only first line validated |
| `strpos` and `preg_match` check different parts | Conditions can be satisfied independently using newline |
| Flag encoded in Brainfuck | Obfuscation only — decoded trivially with online tools |

### Why the `/m` Flag Is Dangerous in Validation

```php
// VULNERABLE — multiline mode allows newline bypass
preg_match('/^-?[a-z0-9]+$/m', $password)

// SECURE — without /m flag, checks entire string as one unit
preg_match('/^-?[a-z0-9]+$/', $password)
```

Without `/m`, the `^` and `$` anchors apply to the **entire string** — a newline in the middle would cause the regex to fail because `R_4r3@` on the second line does not match `[a-z0-9]`. Removing the `/m` flag closes this bypass entirely.

---

# Exploitation Flow (OWASP A05 + A03 Mapping)

1. Attacker visits the website — navigation reveals `/index.php`
2. Dirsearch run — discovers `/flag.php`, `/robots.txt`, `/source.php`
3. `/flag.php` returns empty — flag exists but is not directly displayed
4. `/source.php` reveals the full PHP validation logic
5. `/robots.txt` reveals hidden path `/S3cr3t.php`
6. `/S3cr3t.php` accessed — restricted login page with password field
7. Source code analysed — two contradictory conditions identified
8. `/m` multiline flag spotted in regex — newline bypass technique identified
9. Password crafted: `111%0aR_4r3@` — satisfies both conditions simultaneously
10. Password submitted to `/S3cr3t.php` — browser shows blank response
11. Burp Suite Repeater used — raw response reveals Brainfuck encoded output
12. Brainfuck decoded using dcode.fr → **`FL@g{R3Str1Ct1d_Ar34}`**

---

# Conclusion

This challenge demonstrates a **regex bypass using multiline mode** — a subtle but powerful vulnerability. The developer intended the regex to enforce that the password only contains safe characters, but the `/m` flag changed how `^` and `$` work — allowing a newline to split the input and bypass the character restriction entirely.

Combined with the source code being publicly accessible via `/source.php` and the hidden path being listed in `robots.txt`, all the information needed to craft the bypass was available on the server itself.

The flag name `R3Str1Ct1d_Ar34` — Restricted Area — is the lesson: a restricted area is only as secure as the logic protecting it.

To prevent this type of bypass, applications should:

- **Never expose source code or validation logic** in publicly accessible files
- **Never list sensitive paths in `robots.txt`** — use proper access controls instead
- **Test regex patterns carefully** — the `/m` flag changes anchor behaviour in ways that can create bypasses
- **Use strict server-side validation** — validate the entire input as a single unit, never line by line
- **Avoid security through obscurity** — encoding output in Brainfuck provides no real protection
