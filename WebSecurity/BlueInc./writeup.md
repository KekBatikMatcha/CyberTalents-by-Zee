# Blue Inc — CyberTalents

---

## Challenge Description
**Blue Inc is a new social media website that's still under construction, However it doesn't have registration yet, but if you are interested in seeing our website then you can login with demo/demo.**

## Challenge Link
https://cybertalents.com/challenges/web/blue-inc

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="705" height="417" alt="image" src="https://github.com/user-attachments/assets/94c0aef0-8fbc-4ddc-9230-5762d05aa590" />
</p>

<p align="center">
<img width="673" height="389" alt="image" src="https://github.com/user-attachments/assets/a6d710b7-5f00-4a3b-af7d-2417aaeb1aa9" />
</p>

---

## 2. Challenge Page — Website Home

Upon navigating to the challenge URL, a social media website called **Blue Inc** is presented with the following welcome message:

> **"Welcome to Blue Inc — Next Generation Social Network."**

The page lists the platform's features:
- Post comments and share photographs
- Share links to news or other interesting content
- Play games, chat live, and stream live video
- Control content visibility — public, friends only, or private

<p align="center">
<img width="700" height="419" alt="image" src="https://github.com/user-attachments/assets/6d180f56-9be5-4402-a8c4-e5835e4b5f7f" />
</p>

---

## 3. Exploring the Application — About Page

The **About** page is inspected first. It reveals:

> *"Blue Inc is an American for-profit corporation and an online social media and social networking service based in Menlo Park, California. The Blue Inc website was launched on February 4, 2016, by Mohammed Ahmedberg, along with fellow AUC students and roommates."*

Nothing immediately exploitable here — but it confirms this is a custom-built application, not a known CMS, meaning the vulnerability will likely be in the application logic itself.

<p align="center">
<img width="721" height="413" alt="image" src="https://github.com/user-attachments/assets/3e441ab6-aada-4c6f-a68f-34c1e42ea198" />
</p>

---

## 4. Login Page — Credentials from the Challenge Description

The login page is identified as a potential attack surface. However, re-reading the challenge description reveals a direct clue:

> *"...if you are interested in seeing our website then you can login with **demo/demo**."*

The challenge itself provides a guest account: username `demo`, password `demo`. This is the entry point — not a brute-force or injection attack, but a deliberate low-privilege account to begin the exploration.

<p align="center">
<img width="707" height="413" alt="image" src="https://github.com/user-attachments/assets/636a86e1-5c3a-4f6d-bb33-1c4eabc5ad34" />
</p>

---

## 5. Logging In as `demo` — Session Established

The credentials `demo` / `demo` are entered into the login form. Login succeeds and a session is established.

<p align="center">
<img width="183" height="44" alt="image" src="https://github.com/user-attachments/assets/e3e0d3b8-7f33-495c-a1d7-5e90c2bdc414" />
</p>

The profile page is loaded with the message:

> **"Welcome to your profile demo!"**

<p align="center">
<img width="717" height="404" alt="image" src="https://github.com/user-attachments/assets/985028ed-e067-4bb3-a412-d188e33bd2fb" />
</p>

---

## 6. Inspecting the Request — Cookie and URL Parameter Discovered

With the session active, the browser's **Developer Tools** (Network tab) is used to inspect the HTTP request made when loading the profile page. Two key observations are made:

**1 — The URL contains a GET parameter:**
```
?user=demo
```

**2 — The request headers contain a cookie:**
```
Cookie: user=demo
```

<p align="center">
<img width="646" height="193" alt="image" src="https://github.com/user-attachments/assets/0caf0a45-2349-465b-9f00-a8d922a8cf89" />
</p>

### Why This Is Suspicious

Both the URL parameter and the cookie are set to the string `demo` — the username of the currently logged-in user. The server appears to be using the `user` value directly to determine **which profile to load**, rather than deriving the user identity from a secure, server-side session token.

This pattern is a classic indicator of **Insecure Direct Object Reference (IDOR)** — where a user-controlled value is used to reference a resource (in this case, a user profile), and the server does not validate whether the requester is actually authorised to access that resource.

The question is: what happens if `demo` is changed to `admin`?

---

## 7. Exploiting the IDOR — Changing `user` to `admin`

Using **Burp Suite Repeater**, the intercepted GET request is modified. The `user` parameter in the URL and the `user` cookie are both changed from `demo` to `admin`:

```
GET /?user=admin HTTP/1.1
Cookie: user=admin
```

The modified request is sent to the server.

<p align="center">
<img width="606" height="206" alt="image" src="https://github.com/user-attachments/assets/1bb2c9fe-1160-4b06-acbd-65c219d26928" />
</p>

### What Happened

The server accepted the modified request without any authentication check. It loaded the `admin` profile and returned its contents — including the flag. The server trusted the `user` value supplied by the client without verifying that the requesting session actually belonged to the admin user.

---

## 8. Server Response — Flag Obtained

The server response to the modified request contains:

```
The flag is: 15716a249064f7e9684a816dcdb05282
```

> **FLAG: `15716a249064f7e9684a816dcdb05282`**

---

# OWASP Classification

> **OWASP Top 10: A01 — Broken Access Control**
> The application used a client-supplied `user` parameter to determine which profile to display, without verifying that the authenticated session matched the requested user. An attacker logged in as `demo` could access any other user's profile — including `admin` — simply by changing the parameter value. This is a textbook **Insecure Direct Object Reference (IDOR)** vulnerability.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| Profile loaded based on `user` GET parameter | Any user can request any other user's profile by changing the parameter |
| `user` cookie set to username string | Client-controlled — trivially modified in browser DevTools or Burp Suite |
| No server-side authorisation check | Server does not verify the session matches the requested user |
| Admin profile accessible to guest account | Highest-privilege account exposed to the lowest-privilege user |

### The Correct Fix

```php
// VULNERABLE — trusts client-supplied user value
$username = $_GET['user'];
$profile = getProfile($username);

// SECURE — derives user identity from server-side session only
session_start();
$username = $_SESSION['authenticated_user'];
$profile = getProfile($username);
```

The fix is to **never use client-supplied values to identify the current user**. User identity must be derived exclusively from the server-side session, which the attacker cannot modify. The `user` GET parameter and `user` cookie should be removed entirely — the server already knows who is logged in from the session.

---

# Exploitation Flow (OWASP A01 Mapping)

1. Attacker reads the challenge description — `demo`/`demo` credentials provided
2. Website explored — home page and about page contain no exploitable functionality
3. Login attempted with `demo` / `demo` — access granted
4. Profile page loaded — displays *"Welcome to your profile demo!"*
5. Browser DevTools used to inspect the request — `?user=demo` in URL and `Cookie: user=demo` in headers discovered
6. IDOR identified — server appears to load profile based on client-supplied `user` value
7. Burp Suite Repeater used — `user` parameter and cookie changed from `demo` to `admin`
8. Modified request sent — server returns the admin profile without any authorisation check
9. **FLAG: `15716a249064f7e9684a816dcdb05282`**

---

# Conclusion

This challenge demonstrates an **Insecure Direct Object Reference (IDOR)** vulnerability — one of the most common and impactful access control flaws in web applications. The developer stored the logged-in username in a client-controlled location (a GET parameter and a cookie) and used it directly to fetch profile data. Since the client controls these values, any user can impersonate any other user simply by changing a string.

The challenge design is instructive: the `demo` account is provided openly in the description, making it clear that the initial access is not the challenge. The real challenge is recognising that the session mechanism itself is broken — and that the path from `demo` to `admin` requires nothing more than changing a single value.

To prevent IDOR vulnerabilities, applications must:

- **Never use client-supplied values to identify the authenticated user** — derive identity from the server-side session exclusively
- **Enforce authorisation on every request** — verify the session user matches the requested resource before returning any data
- **Treat all client input as untrusted** — cookies, URL parameters, and headers can all be modified by the attacker
