# Wrong Token - CyberTalents

---

## Challenge Description
**Request to the flag is forbidden due to wrong csrf token ... can you fix it and reveal the flag.**

## Challenge Link
http://cdcamxwl32pue3e6mxwl321pseve6m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="723" height="347" alt="image" src="https://github.com/user-attachments/assets/08479819-87e7-4832-9136-3194acca69bc" />
</p>

---

## 2. Challenge Page

Upon navigating to the challenge URL, the webpage is presented. The description tells us the flag request is blocked due to a **wrong CSRF token** — meaning the application uses a token-based validation system that we need to bypass.

<p align="center">
<img width="959" height="371" alt="image" src="https://github.com/user-attachments/assets/e15a3ed1-6eab-43d3-9446-616e1b9945f8" />
</p>

---

## What is a CSRF Token?

**CSRF (Cross-Site Request Forgery)** is an attack where a malicious website tricks a logged-in user's browser into making unwanted requests to another site. To prevent this, web applications use **CSRF tokens** — unique secret values included in requests to prove the request came from the legitimate site.

### How CSRF Tokens Normally Work

```
1. User visits the page
2. Server generates a unique random token
3. Token is embedded in forms or sent in headers
4. User submits a request — token is included
5. Server checks the token matches — if yes, request is allowed
6. If token is missing or wrong — request is rejected
```

### Why CSRF Tokens Can Be Bypassed

CSRF token validation is only as strong as the server-side check. If the server:
- Accepts any token value without verifying it against a stored secret
- Checks the token using a **loose type comparison** instead of a strict one
- Accepts unexpected data types like `true`, `null`, or `1`

Then the protection can be bypassed without knowing the real token value.

---

## 3. Source Code Analysis — AJAX Request Discovered

Inspecting the page source reveals a JavaScript block making an AJAX POST request to `index.php`:

```javascript
<script type="text/javascript">
    $.ajax('index.php', {
        type: 'post',
        contentType: 'application/json',
        data: '{"action": "view_flag", "_token": "asdjhDJhfkjdI"}',
    });
</script>
```

<p align="center">
<img width="572" height="157" alt="image" src="https://github.com/user-attachments/assets/b8abd469-7d54-4444-a178-ddd3aeacb858" />
</p>

### What Does This Code Do?

| Part | Meaning |
|------|---------|
| `$.ajax('index.php', {...})` | Sends an AJAX request to `index.php` on the server |
| `type: 'post'` | Uses HTTP POST method |
| `contentType: 'application/json'` | Sends data as JSON format |
| `"action": "view_flag"` | Tells the server what action to perform — view the flag |
| `"_token": "asdjhDJhfkjdI"` | The CSRF token being sent — this is the value we need to manipulate |

The application is sending a hardcoded token `"asdjhDJhfkjdI"` — but the challenge description says this token is **wrong**. The server is rejecting it. Our goal is to find a token value that the server accepts.

---

## 4. Intercepting the Request via Burp Suite

The URL is changed to access `index.php` directly:

```
http://cdcamxwl32pue3e6mxwl321pseve6m236mnkmugv0-web.cybertalentslabs.com/index.php
```

The request is intercepted using **Burp Suite** and sent to **Repeater** for analysis. The initial GET request returns no change — because the server expects a POST request with JSON data in the body.


---

## 5. Crafting the Correct Request — Token Bypass

Based on what was discovered in the source code, the request needs to be modified in Burp Suite Repeater to match exactly what the JavaScript sends — but with a manipulated token value.

**Three changes are made to the request:**

<p align="center">
<img width="286" height="137" alt="image" src="https://github.com/user-attachments/assets/1058c4c5-d1fe-4130-85ba-84bf29be7050" />
</p>

**Change 1 — Method changed from GET to POST:**
```
GET /index.php HTTP/1.1  →  POST /index.php HTTP/1.1
```

**Change 2 — Content-Type header set to JSON:**
```
Content-Type: application/json
```

**Change 3 — JSON body added with token changed to `true`:**
```json
{"action": "view_flag", "_token": true}
```


### Why Change the Method to POST?

The source code shows `type: 'post'` in the AJAX request. A GET request to `index.php` would never trigger the flag endpoint — the server only processes the `view_flag` action when it receives a POST request with the correct JSON body.

### Why Set Content-Type to `application/json`?

The source code shows `contentType: 'application/json'`. When PHP or any server-side language receives a request, it reads the `Content-Type` header to know **how to parse the request body**. If `Content-Type` is wrong:

- The server may not parse the JSON body at all
- The `_token` and `action` parameters would never be read
- The request would fail without even checking the token

Setting `Content-Type: application/json` tells the server to parse the body as JSON — allowing it to read both the `action` and `_token` values.

### Why Change `_token` to `true`?

This is the key bypass. The original token is a string `"asdjhDJhfkjdI"` — which the server rejects as wrong. The value `true` is a **boolean** — a completely different data type.

In many server-side languages like PHP, token validation uses **loose type comparison** (`==`) instead of strict type comparison (`===`):

| Comparison | `"asdjhDJhfkjdI" == true` | `"asdjhDJhfkjdI" === true` |
|-----------|--------------------------|---------------------------|
| Result | `true` ✅ | `false` ❌ |
| Why | PHP coerces any non-empty string to `true` in loose comparison | Strict check requires same type AND value |

In PHP loose comparison:
- Any **non-empty string** equals `true`
- Any **non-zero number** equals `true`
- `null`, `""`, `0`, `false` all equal `false`

So when the server checks:

```php
if ($token == true) {
    // show flag
}
```

Sending `"_token": true` as a JSON boolean passes this check — because `true == true` is always true regardless of what the real token value should be.

This is a **Type Juggling vulnerability** — exploiting the difference between loose and strict comparison in PHP.

---

## 6. Flag Obtained

The modified POST request is sent in Burp Suite Repeater. The server validates the token, accepts the request, and returns the flag:

<p align="center">
<img width="290" height="196" alt="image" src="https://github.com/user-attachments/assets/0b6deeff-20ef-4a6d-b845-674bb3304a2b" />
</p>

> **FLAG: `dkjfhsdkhfr43r34r3`**

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A07 — Identification and Authentication Failures**

The CSRF token validation was implemented using loose type comparison, allowing a boolean `true` value to bypass the token check entirely. A properly implemented CSRF token validation should use strict comparison and verify the token against a securely stored server-side value.

---

# Why This Is a Security Issue

The CSRF token protection failed because of a weak server-side validation implementation:

| Issue | Impact |
|-------|--------|
| Token validated using loose comparison `==` | Boolean `true` bypasses any non-empty string check |
| Token hardcoded in client-side JavaScript | Real token value visible in page source |
| No server-side token generation and storage | Server cannot verify the token came from a legitimate session |
| `Content-Type: application/json` accepted without restriction | Attacker can craft exact request body from Burp Suite |

### Loose vs Strict Comparison in PHP

```php
// VULNERABLE — loose comparison
if ($token == true) { ... }        // "anystring" == true → PASSES

// SECURE — strict comparison
if ($token === $expected_token) { ... }  // must match exact value AND type
```

### What a Proper CSRF Token Should Look Like

| Property | Bad Implementation | Good Implementation |
|----------|-------------------|-------------------|
| **Generation** | Hardcoded in source | Random, generated per session server-side |
| **Storage** | Only in client JS | Stored in server session, compared server-side |
| **Comparison** | Loose `==` | Strict `===` |
| **Expiry** | Never expires | Expires with session or after single use |

---

# Exploitation Flow (OWASP A07 Mapping)

1. Attacker accesses the challenge page
2. Page source reveals AJAX POST request to `index.php` with token `"asdjhDJhfkjdI"`
3. Burp Suite intercepts the request to `index.php`
4. Initial GET request returns no flag — server expects POST
5. Request method changed to POST
6. `Content-Type: application/json` header added — server can now parse JSON body
7. JSON body added: `{"action": "view_flag", "_token": "asdjhDJhfkjdI"}` — rejected
8. Token value changed from string `"asdjhDJhfkjdI"` to boolean `true`
9. Server performs loose comparison: `"asdjhDJhfkjdI" == true` → passes
10. Flag revealed: **`dkjfhsdkhfr43r34r3`**

---

# Conclusion

This challenge demonstrates a **PHP Type Juggling vulnerability** in CSRF token validation. The server used loose comparison (`==`) instead of strict comparison (`===`) when checking the token — allowing the boolean value `true` to bypass the check entirely because any non-empty string loosely equals `true` in PHP.

The flag name `Wrong Token` is the lesson — the token itself was not wrong, but the **way the server validated it was wrong**.

To implement CSRF tokens securely, applications should:

- **Generate tokens server-side** using a cryptographically secure random function
- **Store tokens in the session** and compare server-side — never trust client-supplied values
- **Use strict comparison** (`===`) — always verify both value AND data type
- **Expire tokens** after use or when the session ends
- **Never expose the token** in client-side JavaScript source code
