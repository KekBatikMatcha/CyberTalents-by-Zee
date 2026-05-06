# Encrypted Database - CyberTalents

---

## Challenge Description
**The company hired an inexperienced developer, but he told them he hided the database and have it encrypted so the website is totally secure, can you prove that he is wrong?**

## Challenge Link
https://cybertalents.com/challenges/web/encrypted-database

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="662" height="311" alt="image" src="https://github.com/user-attachments/assets/3dd74463-1fd5-494b-9ee1-235bc1e7097a" />
</p>

---

## 2. Challenge Page — TopNewsJournal Website

Upon navigating to the challenge URL, a news journal website called **TopNewsJournal** is presented. It contains a navigation bar with links — **Home, About, Top News, Contact Us, Help** — but clicking through each page reveals nothing obviously exploitable or vulnerable.

<p align="center">
<img width="887" height="361" alt="image" src="https://github.com/user-attachments/assets/6770bf82-9cc1-492e-923d-adb304428c6f" />
</p>

---

## 3. Source Code Analysis — Hidden Directory Discovered

Inspecting the page source reveals the full HTML structure of the page. Two important clues are spotted:

**Clue 1 — Admin assets referenced in the source:**

```html
<link rel="stylesheet" href="admin/assets/style.css">
<script src="admin/assets/app.js"></script>
```

**Clue 2 — Pages loaded via iframe:**

```html
<iframe name="page" src="pages/home.html" style="width:100%; height:600px;border:0;"></iframe>
```

<p align="center">
<img width="930" height="178" alt="image" src="https://github.com/user-attachments/assets/5cf94221-bad2-44cc-a3ba-e0792dcd7870" />
</p>

The reference to `admin/assets/` strongly suggests an **`/admin`** directory exists on the server. This is a common mistake — developers reference admin assets in the public page source, accidentally revealing the existence of a hidden admin panel.

---

## 4. Directory Enumeration — Using Dirsearch

To confirm and discover all hidden directories on the server, the **Dirsearch** tool is used. Dirsearch is a command-line tool that automatically tries thousands of common directory and file names against a target URL to find hidden paths.

### What is Dirsearch?

**Dirsearch** is an open-source web path scanner pre-installed on Kali Linux. It works by sending HTTP requests for common directory and file names — like `/admin`, `/backup`, `/config`, `/login` — and checking the response status codes to see which ones exist.

| Status Code | Meaning |
|-------------|---------|
| `200` | Page exists and is accessible |
| `403` | Page exists but access is forbidden |
| `404` | Page does not exist |
| `301/302` | Redirect — page exists at another location |

The command used:

```bash
dirsearch -u http://cdcamxwl32pue3e6mxmdvww5c1v58m236mnkmugv0-web.cybertalentslabs.com
```

<p align="center">
<img width="478" height="112" alt="image" src="https://github.com/user-attachments/assets/e5666d7a-bce4-4d5b-935f-34355d68011e" />
</p>

The scan returns the following results:

```
[08:38:25] 403 -  571B  - /admin
[08:38:26] 200 -    2KB  - /admin/
[08:38:28] 200 -    2KB  - /admin/index.php
[08:39:29] 403 -  571B  - /pages
[08:39:29] 403 -  571B  - /pages/
```

**Key findings:**

| Path | Status | Meaning |
|------|--------|---------|
| `/admin` | 403 | Exists but direct access blocked |
| `/admin/` | 200 | Accessible with trailing slash |
| `/admin/index.php` | 200 | Admin panel index page is accessible |
| `/pages` | 403 | Exists but access blocked |

The `/admin/` directory is confirmed as publicly accessible.

---

## 5. Accessing the Admin Panel

The URL is changed to access the admin directory directly:

```
http://cdcamxwl32pue3e6mxmdvww5c1v58m236mnkmugv0-web.cybertalentslabs.com/admin/
```

<p align="center">
<img width="950" height="354" alt="image" src="https://github.com/user-attachments/assets/609544fc-34ef-46d9-8785-74247634759d" />
</p>

A **login page for the admin panel** is revealed. However rather than attempting to brute force or bypass the login, the login form itself is inspected for further clues.

---

## 6. Inspecting the Admin Login Form — Hidden Input Field

The admin login page request is intercepted using **Burp Suite** and the response is examined in Repeater. The HTML source of the login form contains a suspicious hidden input field:

```html
<input type="hidden" name="db" value="secret-database/db.json" />
```

<p align="center">
<img width="695" height="349" alt="image" src="https://github.com/user-attachments/assets/8bd1fbe4-d06a-4fd1-a2aa-538bbcd80ae3" />
</p>

### What is a Hidden Input Field?

A `type="hidden"` input field is an HTML form element that is **not visible to the user** in the browser but is still present in the page source and is sent to the server when the form is submitted. Developers use them to pass data to the server silently — but they are fully readable by anyone who inspects the page source.

In this case the hidden field reveals:

- The application is using a **JSON file** as its database: `secret-database/db.json`
- The file path is passed as a form parameter named `db`
- This means the file likely exists at: `/admin/secret-database/db.json`

This is a classic developer mistake — hiding sensitive information in client-side HTML where any user can read it.

---

## 7. Accessing the Database File Directly

The revealed path is appended to the admin URL:

```
http://cdcamxwl32pue3e6mxmdvww5c1v58m236mnkmugv0-web.cybertalentslabs.com/admin/secret-database/db.json
```

<p align="center">
<img width="617" height="59" alt="image" src="https://github.com/user-attachments/assets/18104071-84b3-4d1a-90fb-181e99ddd67c" />
</p>

The file is publicly accessible and returns the full database contents:

<p align="center">
<img width="684" height="352" alt="image" src="https://github.com/user-attachments/assets/e406120c-2b8f-41fe-90bf-f988f8803bf0" />
</p>

```json
"flag":"ab003765f3424bf8e2c8d1d69762d72c"
```

The flag value is present — but it is **hashed** and cannot be submitted directly.

---

## 8. Decrypting the Flag — MD5 Hash Cracking

The flag value `ab003765f3424bf8e2c8d1d69762d72c` is a **MD5 hash** — a one-way cryptographic hash function that converts any input into a fixed 32-character hexadecimal string.

### What is MD5?

**MD5 (Message Digest Algorithm 5)** is a hashing algorithm that produces a 128-bit (32 hex character) hash from any input. It was designed as a one-way function — meaning you cannot mathematically reverse a hash back to its original value.

However MD5 is considered **cryptographically broken** because:

- It is extremely fast to compute — billions of hashes per second on modern hardware
- **Rainbow tables** exist — massive pre-computed databases of common words and their MD5 hashes
- **Online lookup tools** can instantly reverse common MD5 hashes by looking them up in these databases

### What is a Rainbow Table?

A rainbow table is a pre-computed database that maps known input strings to their hash outputs. Instead of cracking the hash mathematically, a lookup tool simply checks if the hash already exists in the database and returns the original input.

| Hash | Original Value |
|------|---------------|
| `ab003765f3424bf8e2c8d1d69762d72c` | `badboy` |

The hash is decrypted using [https://md5decrypt.net/en/](https://md5decrypt.net/en/):

<p align="center">
<img width="690" height="248" alt="image" src="https://github.com/user-attachments/assets/da77e185-cace-49d3-a7d2-bd8dd28a0894" />
</p>

```
ab003765f3424bf8e2c8d1d69762d72c : badboy
```

> **FLAG: `badboy`**

---

# OWASP Classification

This challenge demonstrates multiple vulnerabilities falling under:

> **OWASP Top 10: A01 — Broken Access Control**
> The admin panel and the database file were both accessible without any authentication. The `/admin/` directory and `secret-database/db.json` file should never have been accessible to unauthenticated users.

> **OWASP Top 10: A05 — Security Misconfiguration**
> The developer stored the database path in a client-side hidden input field, stored the database as a plain JSON file inside the web root, and used a weak MD5 hash instead of a proper secure hashing algorithm.

---

# Why This Is a Security Issue

The developer made three compounding mistakes that together exposed the flag:

| Mistake | Impact |
|---------|--------|
| Admin assets referenced in public page source | Reveals existence of `/admin` directory |
| `/admin/` directory publicly accessible | No authentication required to access admin panel |
| Database path stored in hidden HTML input | Any user can read the exact file path from page source |
| Database file stored inside web root | Directly accessible via URL without any authentication |
| Flag stored as MD5 hash | MD5 is broken — reversible instantly with rainbow tables |

### Why MD5 Is Not Secure for Passwords or Flags

| Algorithm | Speed | Salted? | Considered Secure? |
|-----------|-------|---------|-------------------|
| MD5 | Very fast | No | ❌ Broken |
| SHA-1 | Fast | No | ❌ Deprecated |
| bcrypt | Slow (by design) | Yes | ✅ Secure |
| Argon2 | Slow (by design) | Yes | ✅ Recommended |

A secure application should use **bcrypt** or **Argon2** for storing sensitive values — both are designed to be slow and include salting, making rainbow table attacks impractical.

---

# Exploitation Flow (OWASP A01 + A05 Mapping)

1. Attacker accesses the TopNewsJournal website
2. Navigation links show nothing exploitable
3. Page source reveals `admin/assets/` references — `/admin` directory suspected
4. Dirsearch confirms `/admin/` (200) and `/admin/index.php` (200) are accessible
5. Admin login page accessed — login form inspected in Burp Suite
6. Hidden input field reveals database path: `secret-database/db.json`
7. Full path constructed and accessed directly via URL
8. Database file returns: `"flag":"ab003765f3424bf8e2c8d1d69762d72c"`
9. Hash identified as MD5 and cracked using md5decrypt.net rainbow table lookup
10. Flag revealed: **`badboy`**

---

# Conclusion

This challenge proves exactly what the description warns — the developer was wrong to believe the website was secure just because the database was "hidden" and "encrypted." The database was hidden only by obscurity — its path was exposed in the HTML source — and stored in a publicly accessible directory. The MD5 "encryption" provided no real protection since it was cracked instantly using a rainbow table.

True security requires:

- **Never storing sensitive files inside the web root** — databases belong outside the publicly accessible directory
- **Never trusting hidden form fields** — they are fully visible to any user who reads the source
- **Proper access control** — admin panels must require authentication before any content is served
- **Using strong hashing algorithms** like bcrypt or Argon2 instead of MD5 for any sensitive values
- **Security through proper implementation**, not through obscurity
