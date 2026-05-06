# Searching for the Cookies - CyberTalents

---

## Challenge Description
**Simple search website we need to know which cookie to eat ;)**

## Challenge Link
http://cdcamxwl32pue3e6m4m2360mtg301m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="684" height="326" alt="image" src="https://github.com/user-attachments/assets/fa1f2348-5ad2-4873-bf2d-d60474cbb460" />
</p>

---

## 2. Challenge Page — Cookie Search Engine

Upon navigating to the challenge URL, a simple search engine page is presented with a single input field. The description hints that the flag is related to a **cookie** — which immediately suggests we should inspect the browser's cookie storage.

<p align="center">
<img width="808" height="436" alt="image" src="https://github.com/user-attachments/assets/551588d8-3ab0-47fd-ad20-7a3e7a822063" />
</p>

---

## 3. Inspecting the Request Header — Cookie Hint

The page request is intercepted using **Burp Suite**. Examining the request headers reveals an interesting cookie already set by the server:

```
Cookie: try+to+execute+some+js+
```

<p align="center">
<img width="691" height="184" alt="image" src="https://github.com/user-attachments/assets/8f8a1ccf-e1fa-4c83-b1d0-19be58d014dd" />
</p>

The cookie value `try+to+execute+some+js+` is a direct hint from the challenge designer — it is telling us the attack vector is **JavaScript execution**, specifically targeting cookies. This points toward a **Cross-Site Scripting (XSS)** attack using `document.cookie` to read the cookie value.

---

## 4. Source Code Analysis — Obfuscated Script

Inspecting the page source reveals a packed obfuscated JavaScript block using the familiar **Dean Edwards Packer** `eval(function(p,a,c,k,e,d)...)` format:

```javascript
eval(function(p,a,c,k,e,d){...}('...',62,64,'...'.split('|'),0,{}))
```

<p align="center">
<img width="922" height="241" alt="image" src="https://github.com/user-attachments/assets/69f808af-a7b2-4998-8a95-c701945b6991" />
</p>

### What is `eval(function(p,a,c,k,e,d)...)`?

This is **Dean Edwards' JavaScript Packer** — a well-known obfuscation technique. It works by:

1. Taking the original readable JavaScript source code
2. Replacing all words and identifiers with short indexed references
3. Storing all original words in an array
4. At runtime, using `eval()` to reconstruct and execute the original code in the browser

It is **not encryption** — it is purely visual obfuscation. Any decoder tool or JavaScript engine can reverse it instantly.

The script is decoded using [https://thanhle.io.vn/de4js/](https://thanhle.io.vn/de4js/) to reveal the full readable source.

---

## 5. Analysing the Deobfuscated Code

The decoded script reveals a self-executing function with three core parts — cookie management functions, a custom `alert` override, and the flag construction:

```javascript
(function (j, w, d) {
    if (typeof legacyAlert == 'undefined') {
        var legacyAlert = alert
    }

    function createCookie(name, value, days) {
        if (days) {
            var date = new Date();
            date.setTime(date.getTime() + (days * 24 * 60 * 60 * 1000));
            var expires = "; expires=" + date.toGMTString()
        } else var expires = "";
        document.cookie = name + "=" + value + expires + "; path=/"
    }

    function readCookie(name) {
        var nameEQ = name + "=";
        var ca = document.cookie.split(';');
        for (var i = 0; i < ca.length; i++) {
            var c = ca[i];
            while (c.charAt(0) == ' ') c = c.substring(1, c.length);
            if (c.indexOf(nameEQ) == 0) return c.substring(nameEQ.length, c.length)
        }
        return null
    }

    function eraseCookie(name) {
        createCookie(name, "", -1)
    }

    var newAlert = function (p) {
        var f = '';
        f += ([]["fill"] + "")[3];
        f += (true + []["fill"])[10];
        f += (true + []["fill"])[10];
        f += (false + "")[2];
        f += ([]["fill"] + "")[3];
        f += (true + []["fill"])[10];
        f += (true + []["fill"])[10];
        f += (+(20))["to" + String["name"]](21);
        f += ([false] + undefined)[10];
        f += (true + "")[3];
        f += "112";
        eraseCookie('flag');
        createCookie('flag', f, 1);
        legacyAlert(p)
    };
    w.alert = newAlert;
    w.prompt = newAlert;
    w.confirm = newAlert
})(jQuery, window, document);
```

### Code Explanation — Section by Section

**Part 1 — Saving the original alert:**

```javascript
if (typeof legacyAlert == 'undefined') {
    var legacyAlert = alert
}
```

The original browser `alert()` function is saved into `legacyAlert` before it is replaced. This ensures the real alert can still be called later.

**Part 2 — Cookie management functions:**

| Function | Purpose |
|----------|---------|
| `createCookie(name, value, days)` | Creates a browser cookie with a given name, value, and expiry in days |
| `readCookie(name)` | Reads and returns the value of a named cookie from `document.cookie` |
| `eraseCookie(name)` | Deletes a cookie by setting its expiry to -1 day in the past |

**Part 3 — The `newAlert` function and flag construction:**

This is the most important part. When `alert()` is called anywhere on the page, it runs `newAlert` instead. Inside `newAlert`, the flag string `f` is built character by character using JavaScript type coercion:

| Expression | Result | Explanation |
|-----------|--------|-------------|
| `([]["fill"] + "")[3]` | `"c"` | `[].fill` as string = `"function fill() { [native code] }"` → char at index 3 = `"c"` |
| `(true + []["fill"])[10]` | `"o"` | `"true" + fill string` → char at index 10 = `"o"` |
| `(true + []["fill"])[10]` | `"o"` | Same — repeated for second `"o"` |
| `(false + "")[2]` | `"l"` | `"false"` → char at index 2 = `"l"` |
| `([]["fill"] + "")[3]` | `"c"` | Same as first |
| `(true + []["fill"])[10]` | `"o"` | Same |
| `(true + []["fill"])[10]` | `"o"` | Same |
| `(+(20))["to" + String["name"]](21)` | `"k"` | Number 20 converted to base 21 = `"k"` |
| `([false] + undefined)[10]` | `"i"` | `"falseundefined"` → char at index 10 = `"i"` |
| `(true + "")[3]` | `"e"` | `"true"` → char at index 3 = `"e"` |
| `"112"` | `"112"` | Appended as a literal string |

So the constructed flag value is: `c` + `o` + `o` + `l` + `c` + `o` + `o` + `k` + `i` + `e` + `112` = **`coolcookie112`**

**Part 4 — Storing the flag in a cookie:**

```javascript
eraseCookie('flag');
createCookie('flag', f, 1);
legacyAlert(p)
```

When `alert()` is triggered, the function:
1. Deletes any existing `flag` cookie
2. Creates a new `flag` cookie with the constructed value `coolcookie112`
3. Then calls the original `legacyAlert` to show the actual alert popup

**Part 5 — Replacing browser dialog functions:**

```javascript
w.alert = newAlert;
w.prompt = newAlert;
w.confirm = newAlert;
```

All three browser dialog functions — `alert()`, `prompt()`, and `confirm()` — are replaced with `newAlert`. So any XSS payload that calls any of these will trigger the flag cookie to be written.

---

## 6. Testing XSS — Script Tag Injection

A basic XSS payload is first submitted in the search field:

```html
<script>alert</script>
```

<p align="center">
<img width="573" height="150" alt="image" src="https://github.com/user-attachments/assets/ecce5553-41fd-4632-bfe1-4a2a6f3e4818" />
</p>

The result shows the literal text `<script>alert</script>` rendered on the page — the tags were **not executed as HTML** but displayed as plain text. There is also a `'};` error visible in the top left corner of the page.

<p align="center">
<img width="146" height="135" alt="image" src="https://github.com/user-attachments/assets/734a722f-6542-49cf-8170-5bdbb1c6487c" />
</p>

### What Does the `'};` Error Mean?

The `'};` error appearing on the page is a **JavaScript syntax error leak** — it suggests the search input is being embedded **inside** an existing JavaScript string or block in the page. This means the input is inserted into something like:

```javascript
var searchQuery = 'USER_INPUT_HERE';
```

If the user submits `<script>alert</script>`, it becomes:

```javascript
var searchQuery = '<script>alert</script>';
```

The script tags are inside a string — so they are treated as text, not executed. However the fact that the input is inside a JavaScript context means we can **break out of the string** to inject our own code.

---

## 7. Exploiting XSS — Breaking Out of the Script Context

Since the input is embedded inside a JavaScript string, a different approach is needed. The payload used is:

```html
</script><script>alert(document.cookie)</script>
```

<p align="center">
<img width="346" height="120" alt="image" src="https://github.com/user-attachments/assets/1a65b8c1-8896-4f91-bdd5-c2417698b015" />
</p>

### Why `</script>` at the Beginning?

This is the key part of the bypass. When user input is embedded inside a `<script>` block like this:

```html
<script>
    var searchQuery = 'USER_INPUT_HERE';
</script>
```

Submitting `</script>` first **closes the existing script tag** — breaking out of the JavaScript context entirely. Then `<script>alert(document.cookie)</script>` opens a **fresh new script tag** with our own code:

```html
<script>
    var searchQuery = '</script><script>alert(document.cookie)</script>';
</script>
```

The browser parses this as:

1. `<script> var searchQuery = '` — opens script, starts a string
2. `</script>` — closes the script block (browser HTML parser takes priority)
3. `<script>alert(document.cookie)</script>` — new script block executes our code

### What is `document.cookie`?

`document.cookie` is a browser JavaScript property that returns **all cookies stored for the current website** as a single string. When `alert(document.cookie)` is called:

1. It reads all current browser cookies for the page
2. Passes them to `alert()`
3. `alert()` has been replaced by `newAlert` — which builds the flag value and stores it as a new `flag` cookie
4. The alert popup then displays all cookies including the newly created `flag` cookie

---

## 8. Flag Obtained

The alert popup appears showing all cookies including the flag:

```
x=try+to+execute+some+js+; flag=coolcookie112
```

> **FLAG: `coolcookie112`**

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection / Cross-Site Scripting (XSS)**

The search input was embedded directly inside a JavaScript string in the page without proper sanitisation or escaping. This allowed an attacker to break out of the script context using `</script>` and inject arbitrary JavaScript that executed in the browser — reading cookie values and triggering the custom `alert()` override that revealed the flag.

---

# Why This Is a Security Issue

The application embedded user input directly inside a JavaScript `<script>` block without escaping special characters. This created two exploitable conditions:

| Issue | Impact |
|-------|--------|
| Input embedded inside `<script>` block | User can break out using `</script>` |
| No HTML or JavaScript escaping applied | Raw tags and scripts pass through to the browser |
| `alert()` replaced with flag-revealing function | Any `alert()` call triggers the flag cookie |
| `document.cookie` accessible from injected script | All cookie values readable by attacker |

### The Difference Between These Two Injection Points

| Injection Type | Context | Bypass Technique |
|---------------|---------|-----------------|
| Input in HTML body | Tags rendered as HTML | `<script>alert()</script>` directly |
| Input inside `<script>` block | Inside JavaScript string | `</script><script>alert()</script>` to break context |

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the search page
2. Burp Suite reveals cookie already set: `try+to+execute+some+js+` — hints at JavaScript execution
3. Page source reveals packed obfuscated JavaScript
4. Script decoded using de4js — `alert()` replaced with `newAlert` that builds and stores flag as cookie
5. `<script>alert</script>` submitted — rendered as text, not executed
6. `'};` error confirms input is embedded inside a JavaScript string in the page
7. `</script><script>alert(document.cookie)</script>` submitted
8. `</script>` closes the existing script block
9. `<script>alert(document.cookie)</script>` opens new script — executes successfully
10. `alert()` triggers `newAlert` → flag cookie `coolcookie112` created and displayed
11. Flag revealed: **`coolcookie112`**

---

# Conclusion

This challenge demonstrates **Reflected XSS** where user input was embedded inside a JavaScript string context. The standard `<script>` tag injection failed because the input was inside an existing script block — but closing the script tag first with `</script>` provided a clean bypass. The custom `alert()` replacement in the obfuscated source code revealed that triggering any alert would expose the flag through `document.cookie`.

To prevent this vulnerability, applications must:

- **Never embed raw user input inside JavaScript blocks** — always use proper escaping
- **Escape JavaScript special characters** — convert `'`, `"`, `<`, `>`, `/` before inserting into script contexts
- Apply a **Content Security Policy (CSP)** header to restrict inline script execution
- Use the **`HttpOnly` flag** on sensitive cookies — this prevents JavaScript from reading them via `document.cookie`
- Never rely on obfuscation as a security measure — the flag construction logic was fully recoverable from the packed script
