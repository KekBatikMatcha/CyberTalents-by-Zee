# This is Sparta - CyberTalents

---

## Challenge Description
**Morning has broken today they're fighting in the shade when arrows blocked the sun they fell tonight they dine in hell**

**FLAG Format:** `{flagbody}`

## Challenge Link
http://cdcamxwl32pue3e6m4m2360mtg301m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="699" height="355" alt="image" src="https://github.com/user-attachments/assets/cdeaa756-d659-46db-8570-b3064961dc92" />
</p>

---

## 2. Challenge Login Page

Upon navigating to the challenge URL, a login form is presented requiring a **username** and **password**.

A **Hint** button is also available — clicking it reveals the hint: *"easier than ableton"*, suggesting the credentials may be discoverable through client-side analysis.

<p align="center">
<img width="294" height="121" alt="image" src="https://github.com/user-attachments/assets/49458f34-be8f-459a-aa3e-a30a39633e67" />
</p>

---

## 3. Source Code Analysis — Obfuscated JavaScript

Inspecting the page source reveals an obfuscated JavaScript snippet embedded in the application.

The obfuscated code uses hexadecimal string encoding to hide its logic:

```javascript
var _0xae5b=
["\x76\x61\x6C\x75\x65","\x75\x73\x65\x72","\x67\x65\x74\x45\x6C\x65\x6D\x65\x6E\x74\x42\x79\x49\x64","\x70\x61\x73\x73","\x43\x79\x62\x65\x72\x2d\x54\x61\x6c\x65\x6e\x74","
\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x43\x6F\x6E\x67\x72\x61\x74\x7A\x20\x0A\x0A","\x77\x72\x6F\x6E\x67\x20\x50\x61\x73\x
73\x77\x6F\x72\x64"];function check(){var _0xeb80x2=document[_0xae5b[2]](_0xae5b[1])[_0xae5b[0]];var _0xeb80x3=document[_0xae5b[2]](_0xae5b[3])
[_0xae5b[0]];if(_0xeb80x2==_0xae5b[4]&&_0xeb80x3==_0xae5b[4]){alert(_0xae5b[5]);} else {alert(_0xae5b[6]);}}
```

This script handles the login validation entirely on the **client side**, which is a critical vulnerability — the logic and credentials are embedded directly in the browser.

<p align="center">
<img width="945" height="429" alt="image" src="https://github.com/user-attachments/assets/5736633d-0bbd-4549-8be4-ace90550c205" />
</p>

---

## 4. Deobfuscation — Revealing the Credentials

Using the online deobfuscation tool [https://obf-io.deobfuscate.io/](https://obf-io.deobfuscate.io/), the obfuscated JavaScript is decoded into readable form:


<p align="center">
<img width="935" height="326" alt="image" src="https://github.com/user-attachments/assets/d5bc9f3c-3b19-473a-b588-ee742d06d32e" />
</p>

```javascript
var _0xae5b = ["value", "user", "getElementById", "pass", "Cyber-Talent", "Congratz \n\n", "wrong Password"];

function check() {
  var _0xeb80x2 = document[_0xae5b[2]](_0xae5b[1])[_0xae5b[0]];
  var _0xeb80x3 = document[_0xae5b[2]](_0xae5b[3])[_0xae5b[0]];
  if (_0xeb80x2 == _0xae5b[4] && _0xeb80x3 == _0xae5b[4]) {
    alert(_0xae5b[5]);
  } else {
    alert(_0xae5b[6]);
  }
}
```

### Code Explanation

**The Array — `_0xae5b`**

The obfuscator replaced every readable string with an array index lookup instead of writing the strings directly, making the code harder to read at a glance. Once decoded, the array holds:

| Index | Value |
|-------|-------|
| `[0]` | `"value"` |
| `[1]` | `"user"` |
| `[2]` | `"getElementById"` |
| `[3]` | `"pass"` |
| `[4]` | `"Cyber-Talent"` |
| `[5]` | `"Congratz \n\n"` |
| `[6]` | `"wrong Password"` |

**Line by Line Breakdown**

`var _0xeb80x2 = document[_0xae5b[2]](_0xae5b[1])[_0xae5b[0]];`

Substituting the array values:
```javascript
var _0xeb80x2 = document.getElementById("user").value;
```
#### **a) Gets whatever the user typed into the **username field** and stores it in `_0xeb80x2`.**

---

`var _0xeb80x3 = document[_0xae5b[2]](_0xae5b[3])[_0xae5b[0]];`

Substituting:
```javascript
var _0xeb80x3 = document.getElementById("pass").value;
```
#### **b) Gets whatever the user typed into the **password field** and stores it in `_0xeb80x3`.**

---

**The `if` Check:**
```javascript
if (_0xeb80x2 == "Cyber-Talent" && _0xeb80x3 == "Cyber-Talent") {
    alert("Congratz");      // FLAG revealed here
} else {
    alert("wrong Password");
}
```
#### **c) If **both** username AND password equal `"Cyber-Talent"`, the congratulations alert fires and reveals the flag. Otherwise, it shows "wrong Password".
**
**The Full Clean Version:**

Stripping away all obfuscation, the entire function simplifies to:

```javascript
function check() {
  var username = document.getElementById("user").value;
  var password = document.getElementById("pass").value;

  if (username == "Cyber-Talent" && password == "Cyber-Talent") {
    alert("Congratz");
  } else {
    alert("wrong Password");
  }
}
```

**Findings from the deobfuscated code:**

| Field    | Value          |
|----------|----------------|
| Username | `Cyber-Talent` |
| Password | `Cyber-Talent` |

The `check()` function compares the input values directly against the hardcoded string `"Cyber-Talent"` for both fields. Since this validation runs entirely in the browser, the credentials are fully exposed to anyone who inspects the source.

---

## 5. Flag Obtained

Entering the discovered credentials (`Cyber-Talent` / `Cyber-Talent`) into the login form triggers the success condition, and an alert box reveals the flag:

> **FLAG: `{J4V4_Scr1Pt_1S_Aw3s0me}`**

<p align="center">
<img width="447" height="173" alt="image" src="https://github.com/user-attachments/assets/e05f478e-7068-4496-8f93-7ca94702a1bf" />
</p>

---

# Burp Suite Analysis

---

## 6. Intercepting the Login Request

Burp Suite is configured with Proxy Intercept enabled. The challenge is accessed via the Burp embedded browser, and the login request is captured to inspect the HTTP traffic and request headers.

<p align="center">
<img width="938" height="423" alt="image" src="https://github.com/user-attachments/assets/eeae0d39-7e45-465e-9be6-4a9631e6de58" />
</p>

---

## 7. Sending Request to Repeater

The intercepted GET request is forwarded to Burp Suite's **Repeater** tool for further inspection. The Repeater allows resending the request and analysing the server's response to understand how the application behaves.

This confirms that the authentication logic is handled **entirely on the client side** — the server returns the full page including the credentials in the JavaScript, without any server-side validation.

<p align="center">
<img width="710" height="350" alt="image" src="https://github.com/user-attachments/assets/1ebd56b0-85ca-4dbd-903c-c46a72f05d33" />
</p>

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A02 — Cryptographic Failures**

The application relies on JavaScript obfuscation as a substitute for real security. Obfuscation is **not encryption** — it only obscures code visually but does not protect it. Any user who inspects the page source or uses a deobfuscation tool can recover the credentials in seconds.

---

# Why This Is a Security Issue

The application validates credentials **entirely on the client side** using obfuscated JavaScript. This means:

- The credentials (`Cyber-Talent` / `Cyber-Talent`) are embedded directly in the page source
- Obfuscation provides **no real protection** — it is trivially reversed with free online tools
- There is **no server-side authentication** — the server never verifies who is logging in
- Any attacker can bypass the login without even submitting the form

---

# Exploitation Flow (OWASP A02 Mapping)

1. Attacker opens the challenge page in a browser
2. Page source is inspected — obfuscated JavaScript is found
3. Obfuscated code is pasted into a deobfuscation tool
4. Credentials are extracted from the decoded script
5. Credentials are entered into the login form
6. Client-side `check()` function evaluates to `true`
7. Flag is revealed via browser alert

---

# Conclusion

This challenge demonstrates the danger of **client-side authentication** and the false sense of security provided by JavaScript obfuscation. Obfuscation makes code harder to read at a glance, but it is not a security control — it can be reversed instantly with tools like [obf-io.deobfuscate.io](https://obf-io.deobfuscate.io/).

Any sensitive logic such as credential validation **must always be handled server-side**, where the user has no access to the source code or execution environment.
