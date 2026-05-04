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

## 3. Source Code Analysis — Unsanitised URL Parameter

Inspecting the page source reveals a critical piece of PHP code embedded in the page:

```php
<?php
if(!preg_match('/\.\./', $_GET['home'])){
    if(!preg_match('/php\:\/\/input/', $_GET['home'])){
        if (preg_match('/'.basename($_SERVER['PHP_SELF'],'.php').'/', $_GET['home'])){
            include($_GET['home'].'.php');
        }
    }
}
?>
```

<p align="center">
<img width="939" height="329" alt="image" src="https://github.com/user-attachments/assets/251d7cd2-b82c-4123-8826-3ee817ca2d7f" />
</p>

### Why Is This Unsanitised?

The application takes the value from the URL parameter `?home=` and passes it directly into PHP's `include()` function. This means whatever is in the URL gets loaded and executed as a PHP file.

The code does try to apply **3 filters** as weak protection:

| Filter | What it blocks | Why it fails |
|--------|---------------|--------------|
| `preg_match('/\.\./')` | Blocks `../` (directory traversal) | Does NOT block PHP wrappers like `php://filter` |
| `preg_match('/php\:\/\/input/')` | Blocks `php://input` specifically | Does NOT block other wrappers like `php://filter` |
| `preg_match('/index/')` | Only allows values containing the filename | `php://filter/.../resource=index` still contains "index" — passes! |

So the developer tried to block some attacks but left a gap — **`php://filter`** is not blocked and still passes all three checks. This makes the application vulnerable to **Local File Inclusion (LFI) via PHP wrappers**.

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

Because the URL contains the word `index`, it passes the third filter check in the source code — and the `include()` runs with the PHP filter wrapper, returning the entire source code of `index.php` as a Base64 string.

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

---

## 6. Decoding via Online Tool — Flag Obtained

The same Base64 string is also decoded using the online tool [https://www.base64decode.org/](https://www.base64decode.org/), which reveals the full PHP source code of `index.php`.

At the very top of the decoded source, the flag is hardcoded:

```php
<?php

$flag = "{pHp_Wr4P3rs_4r3_Us3fuL}";

?>
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
2. Source code is inspected — `?home=` parameter passed into `include()` is found
3. Three weak `preg_match()` filters are analysed and bypassed
4. `php://filter/convert.base64-encode/resource=index` is injected into the URL
5. The word `index` in the payload passes the third filter check
6. PHP encodes `index.php` as Base64 instead of executing it
7. Long Base64 string is returned in the page output
8. String is decoded via Burp Suite Decoder or base64decode.org
9. Full PHP source code revealed — flag extracted: **`{pHp_Wr4P3rs_4r3_Us3fuL}`**

---

# Conclusion

This challenge demonstrates **Local File Inclusion (LFI)** via **PHP stream wrappers**. The `php://filter` wrapper is a powerful but often overlooked attack vector — it bypasses naive input filters that only block common patterns like `../` or `php://input`, while still allowing an attacker to read sensitive server-side files.

To prevent LFI vulnerabilities, applications should:
- **Never pass user input directly** into `include()`, `require()`, or similar functions
- Use a **whitelist approach** — only allow specific known filenames, never arbitrary input
- **Disable dangerous PHP wrappers** in `php.ini` if not needed (`allow_url_include = Off`)
- Apply proper **input validation** beyond simple pattern matching
