# The Restricted Session - CyberTalents

---

## Challenge Description
**Flag is restricted to logged users only, can you be one of them.**

## Challenge Link
http://cdcamxwl32pue3e6mkm73p97aqzqwm236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="704" height="343" alt="image" src="https://github.com/user-attachments/assets/91be1191-02e4-49da-9c68-ec126285ebea" />
</p>

---

## 2. Challenge Page — Not Logged In

Upon navigating to the challenge URL, the webpage displays:

> **"Welcome to sessions valley**
> **You are not logged-in so you don't have any flag to view**
> **Flag: NULL"**

<p align="center">
<img width="943" height="301" alt="image" src="https://github.com/user-attachments/assets/dbfedb51-47bc-40a7-b144-bcf5a9e00994" />
</p>

The flag is restricted to logged-in users only. The goal is to impersonate a logged-in user without having valid credentials.

---

## What is Session Hijacking?

**Session Hijacking** is an attack where an attacker steals or forges a valid session identifier to impersonate another user without knowing their password.

### How Web Sessions Work

When a user logs into a website, the server creates a **session** and sends a **session ID** to the browser stored as a cookie. On every subsequent request, the browser sends that cookie back to the server — and the server uses it to identify who the user is.

```
User logs in → Server creates session → Server sends PHPSESSID cookie
Browser stores cookie → Every request sends cookie → Server reads cookie → Identifies user
```

### Why Is This Vulnerable?

If an attacker can obtain a valid session ID — whether by finding it exposed, guessing it, or stealing it — they can send it in their own requests and the server will treat them as that legitimate user. The server has no way to distinguish the real user from the attacker if the session ID matches.

| Attack Step | What Happens |
|-------------|-------------|
| Find exposed session IDs | Session store file is publicly accessible |
| Pick a valid session ID | Choose any session from the list |
| Inject it into the request | Replace own cookie with stolen session ID |
| Server validates session | Server thinks attacker is the legitimate user |
| Access granted | Flag revealed |

---

## 3. Source Code Analysis — Session Not Found Message

Inspecting the page source reveals a message:

> **"session not found in data/session_store.txt"**

<p align="center">
<img width="317" height="77" alt="image" src="https://github.com/user-attachments/assets/88227d66-c7a6-44a5-9eaf-fdd084ef29e8" />
</p>

This is a critical information disclosure — the server has accidentally revealed:

- That sessions are stored in a **plain text file** on the server
- The exact **file path**: `data/session_store.txt`
- That the file is potentially **accessible via URL**

---

## 4. Accessing the Session Store File

The file path `data/session_store.txt` is appended directly to the challenge URL:

```
http://cdcamxwl32pue3e6mkm73p97aqzqwm236mnkmugv0-web.cybertalentslabs.com/data/session_store.txt
```

<p align="center">
<img width="592" height="38" alt="image" src="https://github.com/user-attachments/assets/3819fa40-9b66-4664-b417-fc513f155032" />
</p>

The file is publicly accessible and reveals a list of **three valid session IDs**:

<p align="center">
<img width="224" height="160" alt="image" src="https://github.com/user-attachments/assets/27453d06-583a-4dc7-a12d-a275d3731fba" />
</p>

```
1. iuqwhe23eh23kej2hd2u3h2k23
2. 11l3ztdo96ritoitf9fr092ru3
3. ksjdlaskjd23ljd2lkjdkasdlk
```

These are real session IDs belonging to registered users on the server. Any one of these can be used to impersonate that user.

### Why Is Storing Sessions in a Public File Dangerous?

| Issue | Why It Matters |
|-------|---------------|
| File stored inside web root | Anyone can access it via URL |
| No authentication required | No login needed to read the file |
| Plain text format | No encryption or hashing — IDs are directly usable |
| Multiple sessions exposed | All active users are compromised at once |

---

## 5. Intercepting the Request — Source Code Cookie Validation

The page request is intercepted using **Burp Suite**. Inspecting the response reveals a JavaScript snippet in the source code:

```javascript
<script type="text/javascript">
    if(document.cookie !== ''){
        $.post('getcurrentuserinfo.php',{
            'PHPSESSID':document.cookie.match(/PHPSESSID=([^;]+)/)[1]
        },function(data){
            cu = data;
        });
    }
</script>
```

<p align="center">
<img width="737" height="354" alt="image" src="https://github.com/user-attachments/assets/0988d1c5-acaa-49e5-b2fd-1710ec4602be" />
</p>

### What Does This Code Do?

| Part | Meaning |
|------|---------|
| `if(document.cookie !== '')` | Checks if a cookie exists in the browser |
| `document.cookie.match(/PHPSESSID=([^;]+)/)` | Extracts the `PHPSESSID` value from the browser cookie |
| `$.post('getcurrentuserinfo.php', ...)` | Sends a **POST** request to `getcurrentuserinfo.php` with the session ID |
| `function(data){ cu = data; }` | Stores the returned user info in a variable called `cu` |

This tells us the server has a PHP file `getcurrentuserinfo.php` that **accepts a session ID via POST** and returns the user information associated with it. This is the endpoint we can use to verify which session ID belongs to which user.

---

## 6. Querying `getcurrentuserinfo.php` — GET Request

The URL is changed to access `getcurrentuserinfo.php` directly:

```
http://cdcamxwl32pue3e6mkm73p97aqzqwm236mnkmugv0-web.cybertalentslabs.com/getcurrentuserinfo.php
```

<p align="center">
<img width="705" height="332" alt="image" src="https://github.com/user-attachments/assets/45a25372-33a9-42fd-98e3-cb873b1e1b3a" />
</p>

The GET request returns:

```json
{
  "name": null,
  "email": null,
  "session_id": null
}
```

<p align="center">
<img width="260" height="76" alt="image" src="https://github.com/user-attachments/assets/7b06cddb-4744-433f-a158-6e44e8c3150d" />
</p>

All values are `null` because no session cookie was sent with the request. The server cannot identify the user without a valid session ID.

---

## 7. Changing to POST — Injecting a Stolen Session ID

In Burp Suite Repeater, two changes are made to the request:

1. The request method is changed from **GET** to **POST**
2. The `Cookie` header is changed to include one of the stolen session IDs from `data/session_store.txt`:

```
Cookie: PHPSESSID=ksjdlaskjd23ljd2lkjdkasdlk
```

<p align="center">
<img width="701" height="303" alt="image" src="https://github.com/user-attachments/assets/fd49c5b8-3451-4f23-90f4-dc29a3c3fe49" />
</p>

The server responds with the user information associated with that session ID:

```json
{
  "name": "steve",
  "email": "steve@example.com",
  "session_id": "ksjdlaskjd23ljd2lkjdkasdlk"
}
```

This confirms that session ID `ksjdlaskjd23ljd2lkjdkasdlk` belongs to the user **steve**. The server accepted our stolen session ID and returned real user data — the impersonation is working.

---

## 8. Attempting Access — First Try Fails

Back on the main challenge page, the request is intercepted in Burp Suite and the cookie is changed to the stolen session ID. The server responds with:

> **"UserInfo Cookie don't have the username, Validation failed"**

<p align="center">
<img width="698" height="343" alt="image" src="https://github.com/user-attachments/assets/42ea2742-1563-444b-98c3-2b208e311904" />
</p>

This tells us the application requires **more than just the `PHPSESSID` cookie** — it also needs a separate cookie containing the username. The validation checks for a `UserInfo` cookie that must contain the username `steve`.

---

## 9. Adding the UserInfo Cookie — Flag Obtained

The request is modified again in Burp Suite Repeater, this time adding **both** cookies:

```
Cookie: PHPSESSID=ksjdlaskjd23ljd2lkjdkasdlk; UserInfo=steve
```

<p align="center">
<img width="437" height="352" alt="image" src="https://github.com/user-attachments/assets/3a9c0d3d-6f39-4128-9ef5-128d6961e90e" />
</p>

The server now validates both cookies successfully and returns the page as if we are logged in as **steve**:

<p align="center">
<img width="488" height="232" alt="image" src="https://github.com/user-attachments/assets/07ea3e54-7778-4356-9d57-ddc88583d4b7" />
</p>

> **FLAG: `sessionareawesomebutifitsecure`**

---

# OWASP Classification

This vulnerability falls under two OWASP categories:

> **OWASP Top 10: A01 — Broken Access Control**
> The session store file `data/session_store.txt` was stored inside the web root and accessible to anyone without authentication.

> **OWASP Top 10: A07 — Identification and Authentication Failures**
> Valid session IDs were stored in plain text in a publicly accessible file. The application accepted any session ID without verifying whether it was being used by the correct client.

---

# Why This Is a Security Issue

The application had multiple compounding vulnerabilities that together allowed complete authentication bypass:

| Vulnerability | Impact |
|--------------|--------|
| Session file stored in web root | Anyone can read all active session IDs |
| Plain text session storage | No encryption — IDs are directly usable |
| No session binding | Server doesn't verify IP, browser, or other client attributes |
| Separate `UserInfo` cookie with username | Username can be guessed or discovered from `getcurrentuserinfo.php` |
| `getcurrentuserinfo.php` publicly accessible | Attacker can look up which username belongs to any session ID |

---

# Exploitation Flow (OWASP A01 + A07 Mapping)

1. Attacker accesses the challenge page — shown as not logged in
2. Page source reveals `"session not found in data/session_store.txt"`
3. `data/session_store.txt` is accessed directly via URL — three session IDs exposed
4. Page source also reveals `getcurrentuserinfo.php` endpoint and cookie validation script
5. GET request to `getcurrentuserinfo.php` returns null — no session sent
6. Request method changed to POST with stolen session ID in cookie
7. Server returns `{"name":"steve","email":"steve@example.com","session_id":"..."}`
8. Username `steve` is now known
9. Main page request intercepted — only `PHPSESSID` cookie injected → validation fails
10. Both `PHPSESSID` and `UserInfo=steve` cookies injected → validation passes
11. Server returns page as logged-in user steve
12. Flag revealed: **`sessionareawesomebutifitsecure`**

---

# Conclusion

This challenge demonstrates **Session Hijacking** enabled by **insecure session storage** and **broken access control**. The server stored active session IDs in a plain text file inside the publicly accessible web root — effectively handing every active user's identity to any attacker who knew to look.

The flag name itself — `sessionareawesomebutifitsecure` — is the lesson: sessions are a powerful authentication mechanism, but only when implemented securely.

To prevent session hijacking, applications should:

- **Never store session files inside the web root** — keep them in a private server directory
- **Use strong, random, unpredictable session IDs** — never expose them in files
- **Bind sessions to client attributes** such as IP address or User-Agent
- **Use `HttpOnly` and `Secure` flags** on session cookies to prevent JavaScript access and enforce HTTPS
- **Implement server-side session validation** — verify both the session ID and associated user data together, never separately
- **Expire sessions** after a short period of inactivity
