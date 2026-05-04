# Share the Ideas - CyberTalents

---

## Challenge Description
**Can you reveal the admin password?**

## Challenge Link
http://cdcamxwl32pue3e6m4m2360mtg301m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="692" height="350" alt="image" src="https://github.com/user-attachments/assets/4f0ed8c3-ed7e-4267-ae1a-5ef751a65d66" />
</p>

---

## 2. Challenge Page

Upon navigating to the challenge URL, a web application is presented. The page displays:
- A **comment section** showing existing comments
- An **input field** to share a new comment
- A **"Share (you must login)"** button — indicating authentication is required to post
- A **Login** and **Register** link

<p align="center">
<img width="959" height="538" alt="image" src="https://github.com/user-attachments/assets/d96e1207-5db7-4686-8f61-3c1e655b221c" />
</p>

---

## 3. Login Attempt — Failed

An attempt is made to log in without an existing account. The login fails, confirming that a registered account is required to interact with the application.

<p align="center">
<img width="935" height="364" alt="image" src="https://github.com/user-attachments/assets/1845632d-eeb4-48f7-9b19-47dffc3a9396" />
</p>

---

## 4. Registering an Account

A new account is registered via the Register link in order to gain access to the comment feature.

<p align="center">
<img width="541" height="255" alt="image" src="https://github.com/user-attachments/assets/ceece758-4736-4da1-b8eb-39e6d07d71e1" />
</p>

---

## 5. Successful Login & Unsanitised Input Discovered

After logging in, the registered email is displayed confirming authentication was successful. 

<p align="center">
<img width="915" height="280" alt="image" src="https://github.com/user-attachments/assets/6973671d-5cdc-490d-845b-26daf5b87124" />
</p>

A **idea** is then submitted through the input field — and it renders directly on the page **without any sanitisation**.

<p align="center">
<img width="944" height="428" alt="image" src="https://github.com/user-attachments/assets/52d72f11-3e45-4105-888f-2b476543d50f" />
</p>

### What is Sanitisation — and Why Does it Matter?

| | Sanitised Input | Unsanitised Input |
|---|---|---|
| **What it does** | Strips or escapes special characters before processing | Passes raw user input directly into the database query |
| **Example input** | `' OR '1'='1'` → treated as plain text, harmless | `' OR '1'='1'` → interpreted as SQL code, dangerous |
| **Risk** | None — input cannot alter query logic | High — attacker can manipulate or extract database data |

When input is **not sanitised**, whatever the user types is passed directly into the SQL query. This means an attacker can inject SQL code disguised as a comment, tricking the database into executing unintended commands.

---

## 6. Testing SQL Injection — Basic Payload

To confirm the vulnerability, a classic SQL injection payload is submitted in the comment input field:

```sql
' OR '1'='1'
```

**How it works:**

The original query likely looks something like:

```sql
SELECT * FROM comments WHERE input = '[user input]';
```

By injecting `' OR '1'='1'`, the query becomes:

```sql
SELECT * FROM comments WHERE input = '' OR '1'='1';
```

Since `'1'='1'` is always `true`, the `WHERE` condition is bypassed and the query returns data.

---

## 7. Error Confirms SQL Injection — "Unrecognized Token"

The application returns the error:

> **"unrecognized token"**

<p align="center">
<img width="414" height="149" alt="image" src="https://github.com/user-attachments/assets/6ccb2c26-0ea1-474d-a30f-18f7c93222a9" />
</p>

This is actually a **good sign** for the attacker. The error means:
- The input was **not sanitised** — it was passed raw into the SQL query
- The database tried to **parse the injected text as SQL** and threw a syntax error
- This confirms the input field is **injectable** — we just need to refine the payload

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
| `));--` | Closes open brackets and comments out the rest of the original query |

The application returns the version number **`3.8.7.1`** in the comment list, confirming the backend database is **SQLite**. Knowing the database engine is critical — different databases have different syntax, functions, and system tables that need to be used for further exploitation.

<p align="center">
<img width="942" height="311" alt="image" src="https://github.com/user-attachments/assets/a7293b27-e5db-4a27-a5ba-0e4532a49754" />
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
| `sqlite_master` | A special built-in SQLite table that stores the schema of every table in the database |
| `sql` | A column in `sqlite_master` that holds the full `CREATE TABLE` statement for each table |

The application returns:

```sql
CREATE TABLE "xde43_users" (
  "id" int(10) NOT NULL,
  "name" varchar(255) NOT NULL,
  "email" varchar(255) NOT NULL,
  "password" varchar(255) NOT NULL,
  "role" varchar(100) DEFAULT NULL
)
```

This reveals:
- The table name is **`xde43_users`**
- It contains the columns: `id`, `name`, `email`, `password`, and `role`

<p align="center">
<img width="938" height="310" alt="image" src="https://github.com/user-attachments/assets/2f7f9c6c-12da-46e8-9d7b-d6a129b8dff0" />
</p>

### How Do We Know the Role is "admin"?

From the schema, we can see there is a **`role`** column in the `xde43_users` table. This column stores what type of user each account is — for example `"admin"`, `"user"`, `"moderator"`, etc.

We target `role="admin"` because:
- The challenge description asks us to **reveal the admin password** — so we know an admin account exists
- The `role` column in the schema tells us the application uses role-based access control
- `"admin"` is the standard value used to identify administrator accounts in most web applications

So the logic is: **schema → discover `role` column → target `role="admin"` → extract their `password`**.

---

## 10. Extracting the Admin Password — Flag Obtained

With the table structure fully known, the admin password is extracted directly:

```sql
' || (select password from xde43_users where role="admin"));--
```

**How it works:**

| Part | Meaning |
|------|---------|
| `select password` | Retrieves the value in the `password` column |
| `from xde43_users` | From the user table discovered in the schema |
| `where role="admin"` | Filters to return only the admin account's password |

The application returns the admin password in the comment list, which is the flag:

> **FLAG: `flag245698`**

<p align="center">
<img width="937" height="305" alt="image" src="https://github.com/user-attachments/assets/ecca7384-351c-44fd-b90f-6d294b8a2067" />
</p>

<p align="center">
<img width="958" height="286" alt="image" src="https://github.com/user-attachments/assets/57036fd5-386c-4cb9-a5f2-9e024f05a521" />
</p>

---

# Burp Suite Analysis

---

## 11. Replicating the Injection via Burp Suite

The same SQL injection payload is replicated using Burp Suite's **Repeater** tool by intercepting the request when the comment is submitted.

Since the input is sent via a **GET request in the URL**, spaces cannot be used directly — they must be URL-encoded. In URL encoding, **`+`** is a valid substitute for a space character.

So the payload becomes:

```
'+||+(select+password+from+xde43_users+where+role="admin"));--
```

**Why `+` instead of spaces?**

| Character | URL-encoded form | Meaning |
|-----------|-----------------|---------|
| ` ` (space) | `%20` or `+` | Both are valid space representations in a URL |

Using `+` keeps the payload readable while still being correctly interpreted by the server as spaces, allowing the SQL query to execute identically to the browser-based injection.

The modified request is sent via Repeater and the server responds with `flag245698`, confirming the vulnerability is fully exploitable at the HTTP request level — not just through the browser interface.

<p align="center">
<img width="718" height="350" alt="image" src="https://github.com/user-attachments/assets/d2698aff-f98a-4286-be25-a103f61236c1" />
</p>

### Why Is the Input Not Sanitised?

Looking at the application's behaviour, the input is not sanitised because:

1. **Raw input is passed to the query** — the application takes whatever is typed and embeds it directly into the SQL string without escaping special characters like `'`, `"`, or `--`
2. **No parameterised queries** — a secure application would use prepared statements where the query structure is fixed and user input is treated only as data, never as code
3. **Error messages are exposed** — returning raw database errors like "unrecognized token" to the user leaks internal information and confirms injection is possible
4. **No Web Application Firewall (WAF)** — there is no layer filtering or blocking known injection patterns before they reach the database

A properly sanitised application would escape the single quote `'` in the input so that it cannot break out of the SQL string context, making injection impossible.

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection**

SQL Injection occurs when user-supplied input is incorporated into a database query without proper sanitisation or parameterisation, allowing attackers to manipulate the query logic and access unauthorised data.

---

# Why This Is a Security Issue

The application passes user input directly into SQL queries without any sanitisation. This allows an attacker to:

- Confirm injection via database error messages
- Identify the database engine and version
- Extract the full database schema including all table and column names
- Retrieve sensitive data such as admin passwords directly from the database

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the application and registers an account
2. A comment is submitted and appears unsanitised on the page
3. `' OR '1'='1'` payload confirms the injection point via an error
4. `sqlite_version()` identifies the backend as **SQLite 3.8.7.1**
5. `sqlite_master` reveals the full schema including the `xde43_users` table
6. The `role` column is identified — `"admin"` is targeted
7. Admin password is extracted using a targeted `SELECT` query
8. Flag is revealed: **`flag245698`**
9. Exploitation confirmed via Burp Suite Repeater using `+` as URL-encoded spaces

---

# Conclusion

This challenge demonstrates **OWASP A03: Injection**, specifically SQL Injection against a SQLite backend. The absence of input sanitisation and parameterised queries allowed full database enumeration and sensitive data extraction using progressively refined payloads.

To prevent SQL Injection, applications must always use **prepared statements** (parameterised queries), **input validation**, and follow the principle of **least privilege** for database accounts.
