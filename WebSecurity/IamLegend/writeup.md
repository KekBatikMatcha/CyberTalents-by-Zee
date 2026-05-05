# I Am Legend - CyberTalents

---

## Challenge Description
**If I'm a legend, then why am I so lonely.**

**FLAG Format:** `FLAG{}`

## Challenge Link
http://cdcamxwl32pue3e6m4m2360mtg301m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="670" height="392" alt="image" src="https://github.com/user-attachments/assets/b4be03c8-ed4e-44d7-8e19-cfd326d428ea" />
</p>

---

## 2. Challenge Page — Login Form

Upon navigating to the challenge URL, a login page titled **"I Am Legend"** is presented. It contains a username and password input field requiring credentials to proceed.

<p align="center">
<img width="876" height="340" alt="image" src="https://github.com/user-attachments/assets/f147a8dd-28e8-4138-b1cf-3bea2ff9edc7" />
</p>

---

## 3. Source Code Analysis — Suspicious Script Discovered

Inspecting the page source reveals a very unusual block of JavaScript. Instead of readable code, the script is made up entirely of special characters — brackets, plus signs, exclamation marks, and empty arrays:

```
[][[[]+[][+[]]][+[]][++[++[++[++[[]][+[]]][+[]]][+[]]][+[]]]+...
```

<p align="center">
<img width="913" height="392" alt="image" src="https://github.com/user-attachments/assets/79fe8382-55e6-4a71-93d7-b8f2a9457196" />
</p>

At first glance this looks like random noise — but it is actually a real, valid, and executable JavaScript program written in a technique called **JSFuck**.

### What is JSFuck?

**JSFuck** is an esoteric programming style that allows any JavaScript program to be written using only **6 characters**:

```
[ ] ( ) ! +
```

It exploits JavaScript's type coercion system — the way JavaScript automatically converts between data types like numbers, booleans, and strings. By chaining these 6 characters in specific combinations, any string, number, or function call can be constructed without ever using a single letter or digit.

**How does JSFuck work — basic examples:**

| Expression | JavaScript evaluates to | Why |
|------------|------------------------|-----|
| `+[]` | `0` | Empty array coerced to number = 0 |
| `+!+[]` | `1` | `![]` = false, `+false` = 0, `+!+[]` = 1 |
| `[]+[]` | `""` | Two empty arrays concatenated = empty string |
| `![]` | `false` | Empty array is truthy, `!` flips it |
| `!![]` | `true` | Double negation of truthy value |

By combining these building blocks, every character in the alphabet can be constructed, which means any JavaScript code can be written — including functions, `alert()` calls, and `if` statements — using only `[ ] ( ) ! +`.

This makes the code completely unreadable to humans while being perfectly executable by the browser — which is why it is used as an **obfuscation technique**.

---

## 4. Deobfuscation — Decoding the JSFuck Script

The JSFuck script is pasted into the online deobfuscation tool [https://thanhle.io.vn/de4js/](https://thanhle.io.vn/de4js/) which decodes it into readable JavaScript.

> **Note:** Not all JSFuck decoders can handle every variant — this specific tool was found to work for this particular script.

<p align="center">
<img width="800" height="380" alt="image" src="https://github.com/user-attachments/assets/82124552-91bd-4348-a579-eae44de587ef" />
</p>

The decoded output reveals the following JavaScript function:

```javascript
function check() {
    var user = document["getElementById"]("user")["value"];
    var pass = document["getElementById"]("pass")["value"];
    if (user == "Cyber" && pass == "Talent") {
        alert("Congratz \n Flag: {J4V4_Scr1Pt_1S_S0_D4MN_FUN}");
    } else {
        alert("wrong Password");
    }
}
```

### Code Explanation

This is the same client-side authentication pattern seen in previous challenges. Breaking it down:

**The credential check:**

```javascript
var user = document["getElementById"]("user")["value"];
var pass = document["getElementById"]("pass")["value"];
```

These two lines get whatever the user typed into the `user` and `pass` input fields.

**The if statement:**

```javascript
if (user == "Cyber" && pass == "Talent") {
    alert("Congratz \n Flag: {J4V4_Scr1Pt_1S_S0_D4MN_FUN}");
} else {
    alert("wrong Password");
}
```

If both the username equals `"Cyber"` **AND** the password equals `"Talent"`, the browser shows a congratulations alert containing the flag. Otherwise it shows "wrong Password".

**Credentials discovered:**

| Field | Value |
|-------|-------|
| Username | `Cyber` |
| Password | `Talent` |

Since this validation runs entirely in the browser, the credentials are embedded directly in the page — protected only by JSFuck obfuscation, which is trivially reversed with a decoder tool.

---

## 5. Flag Obtained

Entering the discovered credentials into the login form triggers the success alert which reveals the flag:

> **FLAG: `{J4V4_Scr1Pt_1S_S0_D4MN_FUN}`**

Based on the challenge's flag format `FLAG{}`, the complete flag is:

> **FLAG: `FLAG{J4V4_Scr1Pt_1S_S0_D4MN_FUN}`**

<p align="center">
<img width="356" height="150" alt="image" src="https://github.com/user-attachments/assets/1f1590d1-8b00-4927-8490-3c8d4df3ebc6" />
</p>

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A02 — Cryptographic Failures**

The application relies on **JSFuck obfuscation** as a security measure — but obfuscation is not encryption. It does not protect the code from being read, it only makes it harder to read at a glance. Any attacker who recognises the pattern can decode it instantly using freely available online tools, revealing the hardcoded credentials and the flag.

---

# Why This Is a Security Issue

The application performs authentication **entirely on the client side** using obfuscated JavaScript. This means:

- The credentials `Cyber` / `Talent` are embedded directly in the page source
- JSFuck obfuscation provides **no real protection** — it is reversed with a single online tool
- The browser executes the `check()` function locally — the server **never verifies** the credentials
- Any visitor can read the source, decode it, and extract the credentials without even attempting to log in

### JSFuck vs Real Encryption

| | JSFuck Obfuscation | Real Encryption |
|---|---|---|
| **Can it be reversed?** | ✅ Yes — instantly with free tools | Only with the correct key |
| **Is the data hidden?** | ❌ No — just visually scrambled | Yes — mathematically protected |
| **Provides security?** | ❌ None | Yes — when implemented correctly |
| **Used for?** | Code obfuscation, CTF challenges | Protecting passwords, tokens, data |

---

# Exploitation Flow (OWASP A02 Mapping)

1. Attacker navigates to the login page
2. Page source is inspected — a large block of JSFuck is found
3. The unusual `[ ] ( ) ! +` pattern is recognised as JSFuck encoding
4. JSFuck script is pasted into [https://thanhle.io.vn/de4js/](https://thanhle.io.vn/de4js/)
5. Decoded output reveals the `check()` function with hardcoded credentials
6. Username `Cyber` and password `Talent` are extracted from the `if` statement
7. Credentials are entered into the login form
8. Browser executes `check()` and shows the congratulations alert
9. Flag extracted: **`FLAG{J4V4_Scr1Pt_1S_S0_D4MN_FUN}`**

---

# Conclusion

This challenge demonstrates the danger of **client-side authentication** and the false sense of security provided by **JSFuck obfuscation**. While JSFuck makes JavaScript completely unreadable at first glance, it is not a cryptographic protection — it is purely a visual transformation that any decoder tool can reverse in seconds.

The key lesson here is that **any code sent to the browser is accessible to the user**. No matter how obfuscated the JavaScript is — whether using hex encoding (like in "This is Sparta"), array-based obfuscation, or JSFuck — a determined attacker can always recover the original logic.

Authentication and sensitive logic must always be handled **server-side**, where the source code is never exposed to the client.
