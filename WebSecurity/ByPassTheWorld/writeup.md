# Bypass The World — CyberTalents

---

## Challenge Description
**I Don't Care if the world is against you, but i believe that you can bypass the world.**

## Challenge Link
https://cybertalents.com/challenges/web/bypass-the-world

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="689" height="326" alt="image" src="https://github.com/user-attachments/assets/20efa285-948c-46d0-9c09-753f2cfe8bc7" />
</p>

---

## 2. Challenge Page — Login Interface

Upon navigating to the challenge URL, a login page is presented with the message:

> **"☃ Its been too long ,,, Let's Warm up"**

The page contains a username and password input field, along with a **"wanna source?"** link that reveals the PHP source code behind the authentication logic.

<p align="center">
<img width="398" height="322" alt="image" src="https://github.com/user-attachments/assets/76ad7e3e-83b1-45d1-a0cb-72ad9d6312dd" />
</p>

---

## 3. Source Code Analysis — The PHP Validation Logic

Clicking **"wanna source?"** reveals the PHP source code handling the login query:

```php
$name = preg_replace('/\'/', '', $name);
$pass = preg_replace('/\'/', '', $pass);
$query = "SELECT * FROM users where name = '$name' and password = '$pass'";
```

### Code Explanation — Line by Line

**Line 1 — Sanitise the username:**
```php
$name = preg_replace('/\'/', '', $name);
```
All single quote characters (`'`) are stripped from the username input before it is inserted into the query. The developer's intention is to prevent SQL injection by removing the character most commonly used to break out of a string in SQL.

**Line 2 — Sanitise the password:**
```php
$pass = preg_replace('/\'/', '', $pass);
```
The same single quote stripping is applied to the password input.

**Line 3 — Build the SQL query:**
```php
$query = "SELECT * FROM users where name = '$name' and password = '$pass'";
```
Both sanitised values are inserted directly into the SQL query using string concatenation. The query wraps each value in single quotes — so the name becomes `'$name'` and the password becomes `'$pass'`.

### The Security Flaw

The developer removed single quotes to prevent SQL injection — but this sanitisation is **incomplete**. It only removes `'` characters. It does not remove or escape **backslashes** (`\`). This is the key weakness that the bypass exploits.

---

## 4. Understanding the Bypass — Backslash Escaping Attack

### What is a Backslash Escaping Attack

In SQL, a backslash (`\`) is used as an escape character — it tells the database to treat the next character as a literal character rather than a special one. So `\'` means a literal single quote — not the end of a string.

When the username input contains a backslash `\`, it escapes the closing single quote that the SQL query places around `$name`. This causes the quote to become part of the string rather than closing it — which breaks the query structure in a way the developer never anticipated.

### How the Payload Works — Step by Step

**Username field:** `\`
**Password field:** `OR 1=1#`

**Step 1 — Username `\` is inserted into the query:**

The query template is:
```sql
SELECT * FROM users where name = '$name' and password = '$pass'
```

After inserting `\` as the username:
```sql
SELECT * FROM users where name = '\' and password = '$pass'
```

The backslash `\` escapes the closing single quote after `name` — so `\'` becomes a literal quote character inside the string, not the closing delimiter. This means the string is still open — it now extends into the `and password =` part of the query.

**Step 2 — Password `OR 1=1#` is inserted:**

The full query becomes:
```sql
SELECT * FROM users where name = '\' and password = 'OR 1=1#'
```

Which the database actually parses as:

```sql
SELECT * FROM users where name = '\' and password = 'OR 1=1#
```

**Step 3 — The `#` comment closes the query:**

The `#` character is a MySQL comment — everything after it is ignored by the database. So the final parsed query is:

```sql
SELECT * FROM users where name = '\' and password = 'OR 1=1
```

Wait — let me show this more clearly. The database sees the string as split into two logical parts:

| Part | Content | What Database Sees |
|------|---------|-------------------|
| String value of name | `\' and password = ` | A string containing a quote and the AND clause |
| Injected condition | `OR 1=1` | Always-true condition |
| Comment | `#` | Everything after this is ignored |

So the effective query that the database evaluates is:

```sql
SELECT * FROM users where name = '[garbage]' OR 1=1
```

Because `OR 1=1` is always true, the WHERE clause evaluates to true for every row in the database — and the query returns all users, successfully authenticating as the first user in the table (typically `admin`).

### Payload Breakdown Table

| Field | Value | Purpose |
|-------|-------|---------|
| Username | `\` | Backslash escapes the closing quote — breaks the string boundary |
| Password | `OR 1=1#` | Injects always-true condition — `#` comments out the trailing quote |

### Why the Single Quote Filter Failed

| Defence | What It Does | Why It Failed |
|---------|-------------|---------------|
| `preg_replace('/\'/', '', $name)` | Removes `'` from username | Does not remove `\` — backslash still escapes the closing quote |
| `preg_replace('/\'/', '', $pass)` | Removes `'` from password | Password has no single quotes — injection goes through unfiltered |

The filter only removes single quotes. It never considered that a backslash in the username could neutralise the quote that the query itself adds — turning the developer's own query structure against itself.

---

## 5. Submitting the Payload — Flag Obtained

The payload is entered into the login form:

- **Username:** `\`
- **Password:** `` `OR 1=1# ``

<p align="center">
<img width="671" height="215" alt="image" src="https://github.com/user-attachments/assets/30c56240-b401-44ae-9c6b-6818e47e2969" />
</p>

The server processes the manipulated query — the always-true condition causes authentication to succeed — and the flag is revealed:

<p align="center">
<img width="358" height="88" alt="image" src="https://github.com/user-attachments/assets/61530d25-50cc-4c47-ac5c-29f2bc95fd33" />
</p>

> **FLAG: `FLAG{Y0u_Ar3_S0_C00L_T0d4y}`**

---

# OWASP Classification

> **OWASP Top 10: A03 — Injection**
> The application attempted to sanitise SQL injection by stripping single quotes from user input. However, the sanitisation was incomplete — backslash characters were not removed or escaped. An attacker supplied a backslash in the username field to escape the closing quote added by the query template, then injected a classic `OR 1=1` bypass into the password field. The `#` comment character suppressed the remaining SQL syntax, producing an always-true query that bypassed authentication entirely.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| Single quote stripping only — backslash not handled | `\` in username escapes the closing quote in the SQL query |
| String concatenation used to build SQL query | User input directly embedded into the query structure |
| No parameterised queries or prepared statements | The query structure itself can be manipulated by user input |
| `#` comment not filtered | Attacker can comment out trailing SQL to prevent syntax errors |

### The Correct Fix

```php
// VULNERABLE — strips only single quotes, ignores backslash
$name = preg_replace('/\'/', '', $name);
$pass = preg_replace('/\'/', '', $pass);
$query = "SELECT * FROM users where name = '$name' and password = '$pass'";

// SECURE — use prepared statements with parameterised queries
$stmt = $pdo->prepare("SELECT * FROM users WHERE name = ? AND password = ?");
$stmt->execute([$name, $pass]);
$result = $stmt->fetchAll();
```

Prepared statements separate the SQL query structure from the user-supplied data entirely. The database receives the query and the parameters as two separate things — the parameters are never interpreted as SQL syntax regardless of what characters they contain. No amount of backslashes, single quotes, or comment characters can break out of a parameterised query.

### Why Blacklist Filtering Always Fails

| Approach | Example | Bypass |
|----------|---------|--------|
| Strip single quotes | Remove `'` | Use `\` to escape the query's own quotes |
| Strip comments | Remove `#` and `--` | Use `/**/` inline comments instead |
| Strip `OR` | Remove the word `OR` | Use `\|\|` or case variation `oR` |
| Encode input | URL encode | Decode before inserting — still injectable |
| Parameterised queries | `?` placeholders | No bypass possible — data never parsed as SQL |

Every blacklist-based approach has a bypass. The only complete defence is **parameterised queries** — they remove the possibility of injection by design, not by filtering.

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the challenge login page
2. **"wanna source?"** link clicked — PHP source code revealed
3. Source code analysed — single quotes stripped from both fields via `preg_replace`
4. SQL query structure identified — `name = '$name' and password = '$pass'`
5. Backslash escaping technique identified — `\` in username escapes the closing quote
6. Payload crafted — username: `\` — password: `OR 1=1#`
7. Payload submitted — username `\` escapes the closing quote — string boundary broken
8. Password value `OR 1=1` injected into the query as a condition — always evaluates to true
9. `#` comments out the trailing single quote — prevents syntax error
10. Database returns all rows — authentication bypassed
11. **FLAG: `FLAG{Y0u_Ar3_S0_C00L_T0d4y}`**

---

# Conclusion

This challenge demonstrates a **backslash escaping SQL injection bypass** — a technique that defeats single-quote filtering by using the backslash character to escape the closing quote that the SQL query itself provides. The developer correctly identified single quotes as dangerous and removed them — but overlooked the fact that a backslash in the input can neutralise the structural quotes added by the query template itself.

This is a classic example of why **blacklist-based input sanitisation always fails** — there are always edge cases and character combinations that the developer did not anticipate. The backslash is not a SQL injection character on its own — but in combination with the query's own quote character, it becomes one.

The only reliable defence is **parameterised queries** — which do not filter input but instead ensure that input is never parsed as SQL syntax in the first place.

To prevent this type of bypass, applications should:

- **Always use prepared statements** — parameterised queries make injection structurally impossible
- **Never build SQL queries using string concatenation** — user input must never be embedded directly into query strings
- **Never rely on blacklist filtering** — there will always be a bypass that the developer did not think of
- **Use `mysqli_real_escape_string()` as a minimum** if prepared statements cannot be used — it escapes backslashes as well as single quotes
- **Apply the principle of least privilege** — the database user should have only the permissions it needs, limiting the impact of any successful injection
