# Easy Message - CyberTalents

---

## Challenge Description
**I have a message for you.**

## Challenge Link
http://cdcamxwl32pue3e6mxwl322yue3e6m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="713" height="344" alt="image" src="https://github.com/user-attachments/assets/a232b463-3dc2-4c97-9d28-b45322e409b2" />
</p>

---

## 2. Challenge Page — Login Form

Upon navigating to the challenge URL, a simple login page is presented with the title **"Please sign in"** requiring a username and password.

<p align="center">
<img width="893" height="184" alt="image" src="https://github.com/user-attachments/assets/4ac495a9-c073-4e87-8300-accd8f6d5f21" />
</p>

No credentials are provided — the next step is to discover them through enumeration.

---

## 3. Directory Enumeration — Using Dirsearch

To discover hidden files on the server, **Dirsearch** is run against the challenge URL:

```bash
dirsearch -u http://cdcamxwl32pue3e6mxwl322yue3e6m236mnkmugv0-web.cybertalentslabs.com
```

<p align="center">
<img width="374" height="79" alt="image" src="https://github.com/user-attachments/assets/28b3cbad-b6be-483b-86f4-808c4ea87fbe" />
</p>

The scan returns three interesting files:

```
[09:31:30] 200 -    0B  - /db.php
[09:31:46] 200 -  234B  - /index.php.bak
[09:32:17] 200 -   33B  - /robots.txt
```

| File | Size | Notes |
|------|------|-------|
| `/db.php` | 0B | Exists but returns blank — database connection file |
| `/index.php.bak` | 234B | **Backup file of the main application — contains source code** |
| `/robots.txt` | 33B | May contain hidden path references |

---

## 4. Investigating `/db.php` — Blank Response

Navigating to `/db.php` returns a completely blank page. The file exists (status 200) but outputs nothing — this is typical of a database connection file that only sets up a connection without printing anything to the browser. It is included by other PHP files but not meant to be accessed directly.

---

## 5. Downloading `/index.php.bak` — Source Code Exposed

Navigating to `/index.php.bak` downloads the file automatically. This is a **backup file** of `index.php` left behind by the developer — a critical mistake because backup files often contain the full application source code including hardcoded credentials.

<p align="center">
<img width="227" height="102" alt="image" src="https://github.com/user-attachments/assets/82c782b5-3224-48af-a3d9-9eb08e652061" />
</p>

The file contents reveal the full login validation logic:

```php
<?php
$user = $_POST['user'];
$pass = $_POST['pass'];
include('db.php');
if ($user == base64_decode('Q3liZXItVGFsZW50') && $pass == base64_decode('Q3liZXItVGFsZW50'){
    success_login();
}
    else {
        failed_login();
}
?>
```

### Code Explanation — Line by Line

**Reading the submitted credentials:**
```php
$user = $_POST['user'];
$pass = $_POST['pass'];
```
These two lines read the username and password from the POST request body — whatever the user typed into the login form.

**Including the database connection:**
```php
include('db.php');
```
This loads `db.php` which sets up the database connection. The actual database credentials and connection logic are inside that file.

**The credential check:**
```php
if ($user == base64_decode('Q3liZXItVGFsZW50') && $pass == base64_decode('Q3liZXItVGFsZW50')
```

This is the most important line. It compares the submitted username and password against **hardcoded Base64-encoded values**. The function `base64_decode()` decodes the Base64 string at runtime before comparing.

| Part | Meaning |
|------|---------|
| `base64_decode('Q3liZXItVGFsZW50')` | Decodes the Base64 string to its original value |
| `$user == base64_decode(...)` | Checks if submitted username matches the decoded value |
| `$pass == base64_decode(...)` | Checks if submitted password matches the decoded value |
| `&&` | Both conditions must be true simultaneously |
| `success_login()` | Called if both match — grants access |
| `failed_login()` | Called if either does not match — denies access |

**Decoding the credentials:**

Both the username and password use the same Base64 string `Q3liZXItVGFsZW50`:

```
Q3liZXItVGFsZW50  →  Cyber-Talent
```

| Field | Base64 Value | Decoded Value |
|-------|-------------|---------------|
| Username | `Q3liZXItVGFsZW50` | `Cyber-Talent` |
| Password | `Q3liZXItVGFsZW50` | `Cyber-Talent` |

Both credentials are identical — `Cyber-Talent` for both username and password.

### Why Is Hardcoding Credentials in Source Code Dangerous?

The developer used Base64 to obscure the credentials — but Base64 is **not encryption**. It is a reversible encoding that anyone can decode instantly using any online tool or the `base64` command. Storing credentials in source code — even encoded — means:

- Anyone who finds the backup file can recover the credentials immediately
- The credentials can never be changed without modifying the source code
- All instances of the application share the same hardcoded credentials

---

## 6. Checking `/robots.txt` — Another Path Discovered

Navigating to `/robots.txt` reveals:

```
User-agent: *
Disallow: /?source
```

<p align="center">
<img width="196" height="112" alt="image" src="https://github.com/user-attachments/assets/f7ab77de-4c57-4891-99c9-9da03a13c0a3" />
</p>

The `Disallow: /?source` entry is another way to view the source code — navigating to `/?source` in the URL displays the same PHP source code as the downloaded backup file, confirming the credentials once more.

<p align="center">
<img width="548" height="272" alt="image" src="https://github.com/user-attachments/assets/2cb87f93-a64c-46a6-8298-0ea81164a49e" />
</p>

---

## 7. Logging In — Morse Code Message

Using the discovered credentials (`Cyber-Talent` / `Cyber-Talent`), login is successful. The page returns a message in **Morse code**:

```
..-. .-.. .- --. -.--. .. -....- -.- -. ----- .-- -....- -.-- ----- ..- -....- .- .-. ...-- -....- -- ----- .-. ... ...-- -.--.-
```

<p align="center">
<img width="479" height="118" alt="image" src="https://github.com/user-attachments/assets/da6d1051-80fa-4b00-ba13-472282bb7355" />
</p>

### What is Morse Code?

**Morse code** is an encoding system that represents letters and numbers using sequences of dots (`.`) and dashes (`-`). It was originally developed for telegraph communication in the 1830s. In CTF challenges it is commonly used to encode flags as an extra layer of obfuscation.

| Symbol | Meaning |
|--------|---------|
| `.` | Short signal (dot) |
| `-` | Long signal (dash) |
| ` ` (space) | Separator between letters |
| `/` or double space | Separator between words |
| `-.-.--` | Exclamation mark |
| `-.--.-` | Closing parenthesis `)` |
| `-.--.` | Opening parenthesis `(` |

### Decoding the Morse Code

The Morse code is decoded using an online Morse code decoder. Each group of dots and dashes is converted to its corresponding character:

```
..-. = F
.-.. = L
.-  = A
--. = G
-.--. = (
.. = I
-....- = -
-.- = K
-. = N
----- = 0
.-- = W
-....- = -
-.-- = Y
----- = 0
..- = U
-....- = -
.- = A
.-. = R
...-- = 3
-....- = -
-- = M
----- = 0
.-. = R
... = S
...-- = 3
-.--.- = )
```

The decoded message reveals the flag:

> **FLAG: `FLAG(I-KN0W-Y0U-AR3-M0RS3)`**

---

# OWASP Classification

This challenge demonstrates two vulnerabilities:

> **OWASP Top 10: A05 — Security Misconfiguration**
> A backup file `/index.php.bak` was left in the web root and was publicly downloadable — exposing the full application source code including hardcoded credentials. The `robots.txt` file also disclosed a source viewing endpoint `/?source`.

> **OWASP Top 10: A02 — Cryptographic Failures**
> The application stored credentials using Base64 encoding — which is not encryption and provides no security. Anyone who reads the source code can decode the credentials instantly.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| Backup file `/index.php.bak` in web root | Full source code downloadable without authentication |
| Credentials hardcoded in source code | Anyone with source access recovers credentials instantly |
| Base64 used instead of real hashing | Trivially reversible — not a security measure |
| `robots.txt` reveals `/?source` endpoint | Source code viewable via URL — two disclosure paths |
| No server-side session management shown | Login logic entirely in client-accessible backup file |

### Base64 vs Proper Password Storage

| Method | Reversible? | Secure? | Used For |
|--------|------------|---------|---------|
| Base64 | ✅ Yes — instantly | ❌ No | Data encoding only |
| MD5 | ✅ Via rainbow tables | ❌ No — broken | Legacy — never use |
| bcrypt | ❌ No | ✅ Yes | Secure password storage |
| Argon2 | ❌ No | ✅ Yes — recommended | Modern password storage |

---

# Exploitation Flow (OWASP A05 + A02 Mapping)

1. Attacker accesses the login page — no credentials provided
2. Dirsearch run — discovers `/db.php`, `/index.php.bak`, `/robots.txt`
3. `/db.php` returns blank — database connection file, not useful directly
4. `/index.php.bak` downloaded — full PHP source code exposed
5. Source code contains `base64_decode('Q3liZXItVGFsZW50')` for both username and password
6. Base64 decoded → `Cyber-Talent` for both fields
7. `/robots.txt` checked — reveals `/?source` also shows the source code
8. Login with `Cyber-Talent` / `Cyber-Talent` — successful
9. Page returns Morse code message
10. Morse code decoded → **`FLAG(I-KN0W-Y0U-AR3-M0RS3)`**

---

# Conclusion

This challenge demonstrates **Security Misconfiguration** through exposed backup files and **Cryptographic Failure** through the use of Base64 as a credential obfuscation method. The developer left a `.bak` backup file in the web root — downloadable by anyone — which contained the full login source code with credentials encoded only in Base64. Two separate disclosure paths existed: the backup file and the `/?source` endpoint listed in `robots.txt`.

The flag was then encoded in Morse code — an additional layer of obfuscation that is trivially decoded with any online tool.

To prevent these vulnerabilities, applications should:

- **Never leave backup files in the web root** — delete or move them outside the publicly accessible directory
- **Never hardcode credentials in source code** — use environment variables or a secrets manager
- **Use proper password hashing** — bcrypt or Argon2 instead of Base64 or MD5
- **Never list sensitive paths in `robots.txt`** — use proper access controls instead
- **Audit the web root regularly** — remove any `.bak`, `.old`, `.tmp`, or editor swap files before deploying
