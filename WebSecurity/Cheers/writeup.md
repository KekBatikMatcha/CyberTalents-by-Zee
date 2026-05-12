# Cheers - CyberTalents

---

## Challenge Description
**Go search for what cheers you up.**

## Challenge Link
http://cdcamxwl32pue3e6mekgvdqyu93rqm236mnkmugv0-web.cybertalentslabs.com

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="748" height="346" alt="image" src="https://github.com/user-attachments/assets/08ec6100-8a5e-4355-b744-2ac05a15701d" />
</p>

---

## 2. Challenge Page — Error Message Discovered

Upon navigating to the challenge URL, the page immediately displays an error message:

> **"Ops!! Can You Fix This Error For Me Please ??!**
> **Notice: Undefined index: welcome in /var/www/html/index.php on line 14"**

<p align="center">
<img width="542" height="181" alt="image" src="https://github.com/user-attachments/assets/715aa0d2-8624-4fc1-b9b2-3cb8113b58dd" />
</p>

### What Does This Error Mean?

This is a **PHP Notice** — a non-fatal error that PHP outputs when the code tries to access an array index or variable that does not exist. In this case:

```php
// The code is likely doing something like this on line 14:
$value = $_GET['welcome'];
// But the URL has no ?welcome parameter — so PHP throws a Notice
```

| Part | Meaning |
|------|---------|
| `Undefined index: welcome` | The PHP code tried to access `$_GET['welcome']` but `welcome` was not in the URL |
| `/var/www/html/index.php` | The full server path of the PHP file — confirms the web root location |
| `on line 14` | Exact line number where the error occurs |

### Why Is This Error Dangerous?

The server is running with **error display enabled** — meaning PHP errors are shown directly to the user in the browser. This is a critical misconfiguration in production environments because it:

- Reveals the **full server file path** (`/var/www/html/index.php`)
- Reveals **variable names** the code expects (`welcome`)
- Guides the attacker on exactly what parameters to provide
- Acts as a **self-documenting vulnerability** — the application tells you how to use it

In this challenge the error is an intentional hint — but in real applications this would be a serious information disclosure vulnerability.

---

## 3. Fixing the First Error — Adding `?welcome` to the URL

The error tells us the code expects a URL parameter called `welcome`. Adding it to the URL:

```
http://cdcamxwl32pue3e6mekgvdqyu93rqm236mnkmugv0-web.cybertalentslabs.com/?welcome
```

<p align="center">
<img width="520" height="125" alt="image" src="https://github.com/user-attachments/assets/e829ff2e-c263-433d-a23b-8a1f5e0a1fc0" />
</p>

The page now responds with:

> **"Hello !!!**
> **Notice: Undefined index: gimme_flag in /var/www/html/index.php on line 19"**

The first error is resolved — the `welcome` parameter was accepted and the code moved forward to line 19, where it now expects another parameter called `gimme_flag`.

This is the same pattern — the application is revealing its own logic step by step through error messages.

---

## 4. Fixing the Second Error — Adding `gimme_flag` to the URL

The second error tells us the code now also expects a `gimme_flag` parameter. Both parameters need to be in the URL at the same time.

The URL is updated to include both parameters using `&&`:

```
http://cdcamxwl32pue3e6mekgvdqyu93rqm236mnkmugv0-web.cybertalentslabs.com/?welcome&&gimme_flag
```

### Why Use `&&` Between Parameters?

In a URL, multiple GET parameters are normally separated using `&`:

```
?param1=value1&param2=value2
```

In this challenge, the parameters do not need values — just their presence in the URL is enough to satisfy the PHP `isset()` or array access check. Using `&&` or `&` between parameter names without values works because:

| URL Format | Result |
|-----------|--------|
| `?welcome` | `$_GET['welcome']` exists — no error |
| `?welcome&gimme_flag` | Both exist — no errors |
| `?welcome&&gimme_flag` | Also works — the extra `&` is treated as an empty parameter |

The PHP code on the server is likely checking something like:

```php
// Line 14
if (isset($_GET['welcome'])) {
    echo "Hello !!!";
}

// Line 19
if (isset($_GET['gimme_flag'])) {
    echo $flag;
}
```

When both parameters are present in the URL, both `isset()` checks pass and the flag is echoed.

<p align="center">
<img width="259" height="143" alt="image" src="https://github.com/user-attachments/assets/cb8278fd-ca2d-4119-8c0d-515af93ccd65" />
</p>

Both parameters are now satisfied and the flag is revealed:

> **FLAG: `FLAG{k33p_c4lm_st4rt_c0d!ng}`**

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A05 — Security Misconfiguration**

The server had PHP error display enabled in a production environment — outputting detailed error messages including variable names, file paths, and line numbers directly to the browser. This gave the attacker a complete step-by-step guide to satisfying the application's conditions and retrieving the flag.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| PHP errors displayed to browser | Reveals variable names, file paths, and line numbers |
| `Undefined index: welcome` shown | Attacker learns exact GET parameter name required |
| `Undefined index: gimme_flag` shown | Attacker learns second GET parameter name required |
| Full server path exposed | `/var/www/html/index.php` reveals web root structure |
| No authentication required | Flag accessible just by adding URL parameters |

### PHP Error Display — Development vs Production

| Setting | Development | Production |
|---------|-------------|-----------|
| `display_errors = On` | ✅ Useful for debugging | ❌ Never — leaks information |
| `display_errors = Off` | Not needed | ✅ Always use this |
| `log_errors = On` | Optional | ✅ Log to server file instead |
| `error_reporting = E_ALL` | ✅ Show all errors | ❌ Restrict or disable |

In production, errors should be:
- **Logged to a server-side file** that only administrators can access
- **Never displayed** to the browser or end user
- Replaced with a **generic error page** that gives no technical details

---

# Exploitation Flow (OWASP A05 Mapping)

1. Attacker navigates to the challenge URL
2. Page immediately shows PHP error: `Undefined index: welcome`
3. Error reveals the expected GET parameter name — `welcome`
4. URL updated to `/?welcome` — first error resolved
5. New error appears: `Undefined index: gimme_flag` on line 19
6. Second expected parameter `gimme_flag` identified
7. URL updated to `/?welcome&&gimme_flag` — both parameters satisfied
8. Flag revealed: **`FLAG{k33p_c4lm_st4rt_c0d!ng}`**

---

# Conclusion

This challenge demonstrates **Security Misconfiguration** through PHP error display being enabled in production. The application literally guided the attacker step by step — each error message revealed the next parameter name needed, and adding them to the URL was enough to retrieve the flag without any authentication or complex exploitation.

The flag message `k33p_c4lm_st4rt_c0d!ng` — **Keep calm, start coding** — is the lesson: fix your code properly and never expose error details to users.

To prevent this vulnerability, applications should:

- **Always disable `display_errors`** in production PHP configuration
- **Log errors server-side** using `log_errors = On` and a secure log file path
- **Never expose variable names or file paths** in user-facing error messages
- Use a **generic error page** that gives no technical details to the user
- **Require authentication** before any flag or sensitive data is accessible
- Regularly audit `php.ini` settings — `display_errors = Off` must be set in production
