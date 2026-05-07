# Black Inc — CyberTalents

---

## Challenge Description
**Black Inc is a file sharing website, however the file uploads was disabled by an administrator, can you change that or find a bypass?**

## Challenge Link
http://cdcamxwl32pue3e6mxmdvw15h1358m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="683" height="334" alt="image" src="https://github.com/user-attachments/assets/1660ffd6-4925-4d8d-ac62-7208c4744b78" />
</p>

---

## 2. Challenge Page — Website Home

Upon navigating to the challenge URL, a website called **Black Inc** is presented — described as a file sharing service. The page contains a navigation bar with **About Us** and **Login** links, and a file upload section. However, the upload form is not shown — instead an error message is displayed:

> **Error! Sorry, File uploading has been disabled by the administrator!**

<p align="center">
<img width="911" height="386" alt="image" src="https://github.com/user-attachments/assets/60527552-5c16-4ce2-a52d-e9f0544461e7" />
</p>

---

## 3. Exploring the Application — Ruling Out Other Vectors

### About Us
Clicking **About Us** opens a modal popup stating:
> *"Black Inc is a market leader in the file sharing business."*

Nothing exploitable is found here.

<p align="center">
<img width="934" height="402" alt="image" src="https://github.com/user-attachments/assets/98e72fe1-a6a3-4e5a-8283-6f258a959b47" />
</p>

### Login Page — SQL Injection Attempted
The login page is tested using a universal SQL injection payload:

```
' OR '1'='1
```

The injection does not work — the application appears to handle login differently or is not vulnerable to SQL injection. This attack vector is ruled out.

---

## 4. Reviewing the Page Source — Is the Upload Really Disabled?

The HTML source of `index.php` is inspected. The relevant section is:

```html
<div class="alert alert-danger">
    <h4>Error!</h4>
    <p>Sorry, File uploading has been disabled by the administrator!</p>
</div>
```

### Key Observation

There is **no `<form>` tag and no `<input type="file">` element** anywhere in the source. The upload functionality has been completely removed from the HTML — the error message is hardcoded in place of the form.

However, this only means the **front-end UI** is disabled. The **server-side script that handles file uploads** (`index.php`) may still be running and listening for POST requests. The browser UI is not the only way to send data to a server — tools like `curl` can send POST requests directly, bypassing the front-end entirely.

---

## 5. Bypassing the Disabled UI — Using `curl` to Upload Directly

Since the upload form is hidden on the front-end but the server endpoint may still accept file uploads, a `curl` command is used to send a multipart POST request directly to `index.php`:

```bash
curl -F test=@list.txt http://cdcamxwl32pue3e6mxmdvw15h1358m236mnkmugv0-web.cybertalentslabs.com/index.php
```

### Breaking Down the `curl` Command

| Part | Meaning |
|------|---------|
| `curl` | Command-line tool used to send HTTP requests |
| `-F` | Sends the request as a **multipart/form-data** POST — the same format a browser uses when submitting a file upload form |
| `test=` | The **field name** in the form. This is the `name` attribute of the `<input>` field — here it is named `test` |
| `@` | Tells `curl` that what follows is a **file path** to upload, not a plain text value. Without `@`, the value would be sent as a literal string |
| `list.txt` | The local file being uploaded |
| `[url]/index.php` | The server endpoint receiving the upload |

### Why This Works

The browser UI showed an error and hid the form — but that is purely a **front-end restriction**. The server-side PHP script at `index.php` was never actually disabled. By sending a POST request directly with `curl`, the front-end is bypassed entirely and the server processes the upload as normal.

This is a fundamental web security principle: **client-side restrictions can always be bypassed**. Any restriction that only exists in the browser (HTML, JavaScript, CSS) offers no real security. Actual security must be enforced on the server.

---

## 6. Server Response — Flag Obtained

The server responds to the `curl` upload request with:

```html
<p>Here is the flag: 6b768890756adf11a9b6bc3c0f816129</p>
```

<p align="center">
<img width="457" height="211" alt="image" src="https://github.com/user-attachments/assets/bc26412e-c9d1-47bb-9abb-c7f14f8925c6" />
</p>

> **FLAG: `6b768890756adf11a9b6bc3c0f816129`**

---

# OWASP Classification

> **OWASP Top 10: A05 — Security Misconfiguration**
> The file upload functionality was only disabled at the front-end (HTML layer). The server-side handler continued to accept and process file upload requests, meaning the restriction was cosmetic only. Any user who bypassed the browser UI — using `curl`, Burp Suite, or any HTTP client — could still trigger the upload logic and receive the flag.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| Upload form hidden via HTML only | `curl` or any HTTP client bypasses it entirely |
| Server-side handler still active | POST requests to `index.php` are still processed |
| No server-side validation or auth check | Flag returned to any unauthenticated request |
| Client-side restriction mistaken for security | Developer assumed hiding the form = disabling the feature |

### The Correct Fix

```php
// VULNERABLE — only the HTML form was removed, server still accepts uploads
// index.php processes $_FILES regardless

// SECURE — server-side check to actually disable uploads
if (UPLOAD_ENABLED === false) {
    http_response_code(403);
    die('File uploading has been disabled.');
}
```

The fix must live on the **server**, not in the HTML. A proper implementation would check a configuration flag server-side and reject the request before any file handling occurs — regardless of how the request was sent.

---

# Exploitation Flow (OWASP A05 Mapping)

1. Attacker visits the challenge website — sees the upload disabled error message
2. **About Us** modal and **Login** page explored — no other exploitable vectors found
3. SQL injection attempted on login — fails, ruled out
4. Page source inspected — upload form is hidden in HTML, not disabled server-side
5. `curl -F` used to send a multipart POST directly to `index.php`, bypassing the browser UI
6. Server processes the request normally — responds with the flag
7. **FLAG: `6b768890756adf11a9b6bc3c0f816129`**

---

# Conclusion

This challenge demonstrates a **client-side restriction bypass** — one of the most common and fundamental mistakes in web security. The developer disabled the upload form in the HTML, but did not implement any server-side check to reject upload requests.

From a security standpoint, the browser is entirely untrusted. Anything rendered in HTML — forms, buttons, error messages, JavaScript checks — can be ignored or bypassed by an attacker. Real access control must always be enforced on the server, where the attacker has no control.

The lesson: **hiding a feature is not the same as disabling it**.
