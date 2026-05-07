# Who Am I - CyberTalents

---

## Challenge Description
**Do not Start a fight you can not stop it.**

## Challenge Link
https://cybertalents.com/challenges/web/who-am-i

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="671" height="326" alt="image" src="https://github.com/user-attachments/assets/53093568-72cf-4054-b191-2a764b49f42c" />
</p>

---

## 2. Challenge Page — Login Form

Upon navigating to the challenge URL, a login page is presented with the title:

> **"Please Enter Your Username and Password !!"**

The page requires a username and password to proceed.

<p align="center">
<img width="541" height="316" alt="image" src="https://github.com/user-attachments/assets/6a2c4b48-3721-48ac-8d4b-ef5e95f6126c" />
</p>

---

## 3. Source Code Analysis — Guest Credentials Discovered

Inspecting the page source reveals a comment containing hardcoded guest credentials left behind by the developer:

```
Guest Account:
-=-=-=-=-=-=-=-
Username: Guest
Password: Guest
```

<p align="center">
<img width="679" height="355" alt="image" src="https://github.com/user-attachments/assets/d5e4a1a4-8ab5-4e2a-afb8-e46bd886c75d" />
</p>

This is a classic **information disclosure** vulnerability — developers sometimes leave credentials, notes, or debug comments in the page source that are visible to anyone who inspects it. These should never be present in a production application.

---

## 4. Logging In as Guest — Access Denied

Using the discovered credentials (`Guest` / `Guest`), login is successful. However the page immediately shows:

> **"Access Denied. You have no admin privileges, Please login with an administrator account"**

<p align="center">
<img width="793" height="208" alt="image" src="https://github.com/user-attachments/assets/76337c44-0837-4298-b78e-bc8e42e071dc" />
</p>

We are authenticated as a guest but the flag requires admin privileges. The next step is to escalate from guest to admin — without knowing the admin password.

---

## 5. Cookie Inspection — Base64 Encoded Value Found

Opening browser Developer Tools and navigating to **Application → Cookies** reveals a cookie storing the authentication value:

| Cookie Name | Cookie Value |
|-------------|-------------|
| `auth` | `bG9naW49R3Vlc3Q%3D` |

<p align="center">
<img width="812" height="421" alt="image" src="https://github.com/user-attachments/assets/14f9d87d-f802-4f33-8b83-d689fea91c07" />
</p>

The value `bG9naW49R3Vlc3Q%3D` looks like **Base64 encoding** — the mix of letters, numbers, and `=` padding at the end is a strong indicator.

### What is Base64?

**Base64** is an encoding scheme that converts binary data into a text string using 64 printable ASCII characters (A-Z, a-z, 0-9, +, /). It is commonly used to safely transmit data in URLs, cookies, and HTTP headers.

**Key characteristics of Base64:**

- Output always uses only letters, numbers, `+`, `/` and `=` as padding
- The output length is always a multiple of 4 characters
- Ends with `=` or `==` padding characters
- **It is NOT encryption** — it is purely an encoding that anyone can reverse instantly

### Decoding the Cookie

First the URL encoding is reversed — `%3D` is the URL-encoded form of `=`:

```
bG9naW49R3Vlc3Q%3D  →  bG9naW49R3Vlc3Q=
```

Then the Base64 is decoded:

```
bG9naW49R3Vlc3Q=  →  login=Guest
```

The cookie stores the username in plain Base64 encoding — `login=Guest`. The server uses this cookie value to determine who the logged-in user is. Since Base64 is trivially reversible, we can craft any value we want.

---

## 6. Cookie Manipulation — Escalating to Admin

The decoded value `login=Guest` is modified to `login=admin`, then re-encoded back into Base64:

```
login=admin  →  bG9naW49YWRtaW4=
```

**Encoding table:**

| Step | Value |
|------|-------|
| Original cookie | `bG9naW49R3Vlc3Q%3D` |
| URL decoded | `bG9naW49R3Vlc3Q=` |
| Base64 decoded | `login=Guest` |
| Modified value | `login=admin` |
| Re-encoded Base64 | `bG9naW49YWRtaW4=` |

<p align="center">
<img width="317" height="63" alt="image" src="https://github.com/user-attachments/assets/db5a6e90-333e-410a-a095-65ee889d550c" />
</p>

In browser Developer Tools, the cookie value is changed from:

```
bG9naW49R3Vlc3Q%3D
```

to:

```
bG9naW49YWRtaW4=
```

<p align="center">
<img width="608" height="183" alt="image" src="https://github.com/user-attachments/assets/faea443e-ee03-422b-bdbe-e5cc075d5dec" />
</p>

---

## 7. Flag Obtained

After refreshing the page with the modified cookie, the server reads `login=admin` from the cookie and grants admin access. The flag is revealed:

> **FLAG: `FLag{B@D_4uTh1Nt1C4Ti0n}`**

The flag name itself — `B@D_4uTh1Nt1C4Ti0n` — is the lesson: **Bad Authentication**.

---

## Alternative Method — Using Burp Suite

The same cookie manipulation can be done directly in **Burp Suite** without needing to use browser Developer Tools.

### Step 1 — Intercept the Login Request

After logging in as `Guest`, enable **Burp Suite Proxy Intercept** and refresh the page. The intercepted GET request will show the cookie in the request header:

```
GET / HTTP/1.1
Host: [challenge-url]
Cookie: auth=bG9naW49R3Vlc3Q%3D
```

### Step 2 — Send to Repeater

Forward the intercepted request to **Burp Suite Repeater** for modification.

### Step 3 — Decode the Cookie

In Burp Suite, highlight the cookie value `bG9naW49R3Vlc3Q%3D` and right click → **Send to Decoder**. In the Decoder tab:

- First decode as **URL** → `bG9naW49R3Vlc3Q=`
- Then decode as **Base64** → `login=Guest`

### Step 4 — Encode the New Value

In the Decoder tab, type `login=admin` and:

- Encode as **Base64** → `bG9naW49YWRtaW4=`

### Step 5 — Modify the Cookie in Repeater

Back in Repeater, replace the cookie value with the newly encoded admin value:

```
Cookie: auth=bG9naW49YWRtaW4=
```

### Step 6 — Send the Request

Click **Send** in Repeater. The server reads `login=admin` from the modified cookie and the response returns the page with admin access and the flag:

> **FLAG: `FLag{B@D_4uTh1Nt1C4Ti0n}`**

### Why Use Burp Suite Instead of Browser DevTools?

| | Browser DevTools | Burp Suite |
|--|-----------------|-----------|
| **Decode/Encode** | Requires separate online tool | Built-in Decoder tab handles everything |
| **Request control** | Limited — only cookie editor | Full request modification including headers and body |
| **Repeatable** | Manual each time | Save and resend requests easily |
| **Best for** | Quick cookie edits | Full request manipulation and analysis |

Both methods achieve the same result — Burp Suite just gives more control and keeps everything in one tool.

---

# OWASP Classification

This challenge demonstrates two vulnerabilities:

> **OWASP Top 10: A02 — Cryptographic Failures**
> The application stored the authentication value using Base64 encoding — which provides zero security. Base64 is an encoding scheme, not encryption. Anyone can decode and re-encode any value instantly.

> **OWASP Top 10: A07 — Identification and Authentication Failures**
> The server trusted the client-supplied cookie value without any server-side verification. The role and identity of the user was determined entirely by a client-controlled cookie that could be freely modified.

---

# Why This Is a Security Issue

The application made two critical mistakes that together allowed complete privilege escalation:

| Vulnerability | Impact |
|--------------|--------|
| Guest credentials in page source | Anyone can log in without being a real user |
| Authentication stored in client-side cookie | User controls their own identity |
| Base64 used instead of real security | Any user can decode, modify, and re-encode the cookie |
| No server-side session validation | Server trusts cookie value without checking a session store |
| No signature or integrity check on cookie | Modified cookie cannot be detected |

### Base64 vs Real Security

| | Base64 | Proper Session Token |
|--|--------|---------------------|
| **Can it be decoded?** | ✅ Yes — instantly by anyone | ❌ No — random opaque token |
| **Can it be modified?** | ✅ Yes — re-encode anything | ❌ No — server validates against session store |
| **Provides security?** | ❌ None | ✅ Yes — when implemented correctly |
| **Used for?** | Encoding data for transport | Identifying authenticated sessions |

### How a Secure Cookie Should Work

A secure application should never store the username or role directly in a cookie — even encrypted. Instead it should:

1. Generate a **random session ID** (e.g. `a3f9b2c1d4e5...`) on login
2. Store the session ID in a **server-side session store** mapping it to the user
3. Send only the random session ID to the client as the cookie
4. On every request, look up the session ID in the server store to get the real user identity

This way, even if an attacker intercepts or modifies the cookie, they cannot change their identity because the server controls the mapping.

---

# Exploitation Flow (OWASP A02 + A07 Mapping)

1. Attacker accesses the login page
2. Page source inspected — guest credentials `Guest`/`Guest` found in HTML comment
3. Login with `Guest`/`Guest` — authenticated but access denied, admin required
4. Browser DevTools → Application → Cookies reveals auth cookie value
5. Cookie value `bG9naW49R3Vlc3Q%3D` identified as Base64
6. URL decode `%3D` → `=`, then Base64 decode → `login=Guest`
7. Value modified to `login=admin`
8. `login=admin` Base64 encoded → `bG9naW49YWRtaW4=`
9. Cookie value replaced in browser DevTools
10. Page refreshed — server reads `login=admin` and grants admin access
11. Flag revealed: **`FLag{B@D_4uTh1Nt1C4Ti0n}`**

---

# Conclusion

This challenge demonstrates **insecure authentication** through client-side cookie manipulation. The server stored the user's role and identity in a Base64-encoded cookie — but Base64 is not encryption, it is just encoding. Any attacker who inspects their cookie can decode it, change the value, re-encode it, and trick the server into believing they are an admin.

Combined with the guest credentials being exposed in the page source, the application had no real authentication barrier at all.

The flag name says it all — **B@D_4uTh1Nt1C4Ti0n**. Good authentication never trusts the client to tell the server who they are.
