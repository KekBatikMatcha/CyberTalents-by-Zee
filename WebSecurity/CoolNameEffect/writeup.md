# Cool Name Effect - CyberTalents

---

## Challenge Description
**Webmaster developed a simple script to do cool effects on your name, but his code not filtering the inputs correctly execute javascript alert and prove it.**

## Challenge Link
http://cdcamxwl32pue3e6m4m2360mtg301m236mnkmugv0-web.cybertalentslabs.com/

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="680" height="317" alt="image" src="https://github.com/user-attachments/assets/945ebf0f-0532-42a8-9a79-e752974db5ec" />
</p>

---

## 2. Challenge Page — Name Input Form

Upon navigating to the challenge URL, a simple web page is presented with an input field asking for a name. After submitting a name, the text is displayed on the page with a **curved arc animation effect** applied to each letter.

<p align="center">
<img width="886" height="293" alt="image" src="https://github.com/user-attachments/assets/ff05449a-c914-4e0b-83ff-141ecfeefb17" />
</p>

The challenge description hints that the input is **not filtered correctly** — meaning whatever is typed into the input field is rendered directly onto the page without sanitisation, making it a potential **Cross-Site Scripting (XSS)** target.

---

## What is Cross-Site Scripting (XSS)?

**Cross-Site Scripting (XSS)** is a web vulnerability that occurs when an application takes user-supplied input and includes it in a web page without properly escaping or sanitising it. This allows an attacker to inject malicious JavaScript code that the browser then executes.

### Types of XSS

| Type | How It Works | Persists? |
|------|-------------|-----------|
| **Reflected XSS** | Input is reflected back in the same response | No — only affects the current request |
| **Stored XSS** | Input is saved to a database and shown to all visitors | Yes — affects every visitor |
| **DOM-based XSS** | Input is processed by client-side JavaScript and written to the DOM | Depends on implementation |

In this challenge the vulnerability is **Reflected XSS** — the name input is taken and immediately rendered into the page HTML without sanitisation.

### Why Is XSS Dangerous?

- Allows attackers to **execute arbitrary JavaScript** in another user's browser
- Can be used to **steal session cookies** and hijack accounts
- Can **redirect users** to malicious websites
- Can **modify page content** to display fake forms or messages
- In this CTF context — it is used to trigger the flag reveal

---

## 3. Source Code Analysis — Obfuscated Script

Inspecting the page source reveals a heavily obfuscated JavaScript block using the **`eval` + packed** obfuscation format — a common technique using the `p,a,c,k,e,d` function pattern:

```javascript
eval(function(p,a,c,k,e,d){...}('...',62,200,'...'.split('|'),0,{}));
```

<p align="center">
<img width="944" height="396" alt="image" src="https://github.com/user-attachments/assets/1d77df1b-3775-4769-bbd2-3c55decf2afa" />
</p>

### What is `eval(function(p,a,c,k,e,d)...)`?

This is a well-known JavaScript obfuscation technique called **Dean Edwards' Packer**. It works by:

1. Taking the original JavaScript source code
2. Replacing all identifiable words and tokens with short indexed references
3. Storing all the original words in an array
4. At runtime, using `eval()` to reconstruct and execute the original code

It is not encryption — it is just compression and obfuscation. Any JavaScript engine (and any decoder tool) can reverse it instantly.

The script is decoded using the online tool [https://thanhle.io.vn/de4js/](https://thanhle.io.vn/de4js/), which reveals the full readable JavaScript source.

---

## 4. Analysing the Deobfuscated Code — The Flag Section

The decoded script contains two main parts. The first part is the **Arctext jQuery plugin** — a legitimate open-source library that creates the curved letter animation effect on the name input. This part is not malicious.

The **second and more interesting part** is a self-executing function at the bottom:

```javascript
(function (j, w) {
    var legacyAlert = alert;
    var newAlert = function () {
        var z = ['y', 'o', 'u', 'r', ' ', 'f', 'l', 'a', 'g', ' ', 'i', 's', ':'];
        var f = ([]["fill"] + "")[3];
        f += ([false] + undefined)[10];
        f += (NaN + [Infinity])[10];
        f += (NaN + [Infinity])[10];
        f += (+(211))["to" + String["name"]](31)[1];
        f += ([]["entries"]() + "")[3];
        f += (+(35))["to" + String["name"]](36);
        legacyAlert(z.join('') + f)
    };
    w.alert = newAlert;
    w.prompt = newAlert;
    w.confirm = newAlert;
    j(function () {
        $x = j('#name');
        $x.arctext({ radius: 400 });
        $x.arctext('set', { radius: 140, dir: 1 })
    })
})(jQuery, window);
```

### Code Explanation — Line by Line

**Step 1 — Hijacking the alert function:**

```javascript
var legacyAlert = alert;
```

The original browser `alert()` function is saved into `legacyAlert` before it gets replaced. This is done so the real alert can still be called later.

```javascript
w.alert = newAlert;
w.prompt = newAlert;
w.confirm = newAlert;
```

The browser's built-in `alert()`, `prompt()`, and `confirm()` functions are all **replaced** with the custom `newAlert` function. This means any call to `alert()` on this page — whether from the attacker's XSS payload or from the page itself — will trigger `newAlert` instead of the normal alert box.

**Step 2 — Building the prefix string:**

```javascript
var z = ['y', 'o', 'u', 'r', ' ', 'f', 'l', 'a', 'g', ' ', 'i', 's', ':'];
```

This array spells out `"your flag is:"` character by character. Later, `z.join('')` joins them into a single string.

**Step 3 — Building the flag value character by character:**

This is the clever part. The flag string `f` is constructed one character at a time using JavaScript's type coercion:

| Expression | Result | Explanation |
|-----------|--------|-------------|
| `([]["fill"] + "")[3]` | `"c"` | `[].fill` is a function, converting it to string gives `"function fill() { [native code] }"` — character at index 3 is `"c"` |
| `([false] + undefined)[10]` | `"i"` | `[false] + undefined` = `"falseundefined"` — character at index 10 is `"i"` |
| `(NaN + [Infinity])[10]` | `"y"` | `NaN + [Infinity]` = `"NaNInfinity"` — character at index 10 is `"y"` |
| `(NaN + [Infinity])[10]` | `"y"` | Same as above — repeated to get the second `"y"` |
| `(+(211))["to" + String["name"]](31)[1]` | `"p"` | Converts the number 211 to base 31 = `"6p"` — character at index 1 is `"p"` |
| `([]["entries"]() + "")[3]` | `"j"` | `[].entries()` is an iterator, converting to string gives `"[object Array Iterator]"` — character at index 3 is `"j"` |
| `(+(35))["to" + String["name"]](36)` | `"z"` | Converts the number 35 to base 36 = `"z"` |

So the flag characters spell out: `c` + `i` + `y` + `y` + `p` + `j` + `z` = **`ciyypjz`**

The full message shown when `alert()` is called becomes:

> **`your flag is: ciyypjz`**

**Why build it this way?**

The flag is never written as a plain string anywhere in the code — it is **constructed at runtime** from JavaScript type coercion tricks. This makes it much harder to spot by simply searching the source code for the flag value.

**Step 4 — Applying the arc effect to the name input:**

```javascript
j(function () {
    $x = j('#name');
    $x.arctext({ radius: 400 });
    $x.arctext('set', { radius: 140, dir: 1 })
})
```

This finds the HTML element with `id="name"` — the name input display — and applies the curved arc text animation to it using the Arctext jQuery plugin.

---

## 5. Testing XSS — Script Tag Injection

A classic XSS payload is submitted in the name input field:

```html
<sCrRipt>alert(1)</sCriPt>
```

Mixed case is used in the tag name (`<sCrRipt>`) to attempt to bypass any basic filters that check for lowercase `<script>`.

<p align="center">
<img width="646" height="245" alt="image" src="https://github.com/user-attachments/assets/118ee650-2c16-4909-96f5-15fad029ac51" />
</p>

The result shows `alert(1)` displayed as curved text on the page — but **the script tags themselves are not shown**. This means:

- The `<script>` tags were **stripped or not rendered** as HTML tags
- However, the text content `alert(1)` was still output to the page
- This confirms the application is rendering user input into the page HTML — just with some tag filtering

This is still a strong indicator of an XSS vulnerability — the filter is incomplete.

---

## 6. Exploiting XSS — Event Handler Injection & Flag Obtained

Since `<script>` tags are filtered, an alternative XSS vector is tried using an **HTML event handler**:

```html
<img src=test onerror="alert()">
```

**How this works:**

| Part | Meaning |
|------|---------|
| `<img src=test>` | Creates an image element with an invalid `src` value `test` |
| `onerror="alert()"` | When the image fails to load (because `test` is not a real image), the `onerror` event fires |
| `alert()` | Calls the browser's `alert()` function — which has been replaced by `newAlert` |

Because the image source `test` does not exist, the browser immediately triggers the `onerror` event — which calls `alert()` — which has been replaced by the custom `newAlert` function — which builds and displays the flag.

<p align="center">
<img width="362" height="140" alt="image" src="https://github.com/user-attachments/assets/8156082e-f48d-43c8-8091-e9ee33c51586" />
</p>

The alert popup appears showing:

> **`your flag is: ciyypjz`**

> **FLAG: `ciyypjz`**

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A03 — Injection / Cross-Site Scripting (XSS)**

The application renders user-supplied input directly into the page HTML without proper sanitisation. While `<script>` tags appear to be filtered, **HTML event handlers** like `onerror` are not blocked — allowing JavaScript execution through an alternative attack vector.

---

# Why This Is a Security Issue

The application fails to properly sanitise user input before rendering it in the HTML. This means:

- User input is embedded directly into the page DOM
- The `<script>` tag filter is **incomplete** — event handler attributes like `onerror`, `onclick`, and `onload` are not blocked
- Any JavaScript injected via event handlers **executes in the victim's browser**
- In this case the `alert()` function was replaced with a custom function that reveals the flag — but in a real attack it could steal cookies or redirect users

### `<script>` Tags vs Event Handlers

| Attack Vector | Example | Blocked Here? |
|--------------|---------|---------------|
| Script tags | `<script>alert(1)</script>` | ✅ Yes — stripped |
| Event handlers | `<img onerror="alert()">` | ❌ No — executes |
| Inline handlers | `<svg onload="alert()">` | ❌ Likely no |

Blocking only `<script>` tags while allowing event handler attributes is a **partial and ineffective defence**.

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the name input page
2. Page source is inspected — a packed obfuscated JavaScript block is found
3. Script is decoded using [https://thanhle.io.vn/de4js/](https://thanhle.io.vn/de4js/)
4. Decoded code reveals `alert()` has been replaced with `newAlert` which builds and displays the flag
5. `<sCrRipt>alert(1)</sCriPt>` is injected — script tags are stripped but text renders
6. Confirms user input reaches the HTML — XSS vulnerability is present
7. `<img src=test onerror="alert()">` is injected — event handler bypasses the filter
8. Image fails to load → `onerror` fires → `alert()` called → `newAlert` executes
9. Flag revealed: **`ciyypjz`**

---

# Conclusion

This challenge demonstrates **Reflected Cross-Site Scripting (XSS)** where a partial input filter blocked `<script>` tags but failed to sanitise HTML event handler attributes. The `onerror` attribute on an `<img>` tag provided a clean bypass — triggering the custom `alert()` replacement that revealed the flag character by character using JavaScript type coercion.

To prevent XSS, applications must:

- **Never insert raw user input into HTML** without escaping it first
- Use **HTML entity encoding** — convert `<` to `&lt;`, `>` to `&gt;`, `"` to `&quot;` etc.
- Apply a **Content Security Policy (CSP)** header to restrict what scripts can execute
- Use a **proper allowlist** approach — block everything by default and only allow safe characters
- Never rely on **blocklist filters** alone — there are always bypass techniques
