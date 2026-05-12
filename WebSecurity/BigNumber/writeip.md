# Big Number — CyberTalents

---

## Challenge Description
**A big number is needed to save the world.**

## Challenge Link
https://cybertalents.com/challenges/web/big-number

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="694" height="319" alt="image" src="https://github.com/user-attachments/assets/350a5956-1fc1-4714-bce0-b5b41f494297" />
</p>

---

## 2. Challenge Page — Password Input Form

Upon navigating to the challenge URL, a simple web page is presented with the following prompt:

> **"Help The World With a Big Number"**
> **"You can help the world by providing 3 numbers can exceed 999 in value:"**

The page contains a single password input field and a link labelled **"wanna source?"** which reveals the PHP source code behind the challenge.

<p align="center">
<img width="596" height="277" alt="image" src="https://github.com/user-attachments/assets/5aed3f79-da08-4d94-9da3-46dc87801d66" />
</p>

The challenge description and the page prompt are the first clues — the password must be a number, must be greater than 999, but must also be fewer than 4 characters long. This appears contradictory at first glance.

---

## 3. Source Code Analysis — The PHP Validation Logic

Clicking **"wanna source?"** reveals the full PHP source code validating the password input:

```php
<?php if (isset($_GET['password'])) {
    if (is_numeric($_GET['password'])){
        if (strlen($_GET['password']) < 4){
            if ($_GET['password'] > 999)
                die($flag);
            else
                print 'Too little';
        } else
                print 'Too long';
    } else
        print 'Password is not numeric';
}
?>
```

<p align="center">
<img width="226" height="235" alt="image" src="https://github.com/user-attachments/assets/34a30b5c-52c9-4a19-8eed-574d7d61607c" />
</p>

### Code Explanation — Line by Line

**Check 1 — Must be numeric:**
```php
if (is_numeric($_GET['password']))
```
The input must pass PHP's `is_numeric()` check. This function returns `true` for integers, floats, and — critically — **scientific notation strings** like `9e9`.

**Check 2 — Must be fewer than 4 characters long:**
```php
if (strlen($_GET['password']) < 4)
```
The string length of the input must be less than 4 characters. This means values like `1000`, `9999`, or `99999` are all rejected because they are 4 or more characters long.

**Check 3 — Must be greater than 999:**
```php
if ($_GET['password'] > 999)
    die($flag);
```
The numeric value of the input must be greater than 999. Any value of 1000 or above reveals the flag.

### The Contradiction

| Condition | Requirement |
|-----------|-------------|
| `is_numeric()` | Input must be a number |
| `strlen() < 4` | Input must be 3 characters or fewer |
| `> 999` | Numeric value must exceed 999 |

The smallest integer greater than 999 is `1000` — which is 4 characters long. So no plain integer can satisfy all three conditions simultaneously. This is where **PHP scientific notation** becomes the bypass.

---

## 4. Understanding the Bypass — Scientific Notation in PHP

### What is Scientific Notation

Scientific notation is a way of expressing very large or very small numbers using a base number and an exponent. The format is:

```
[base]e[exponent]
```

For example:

| Scientific Notation | Equivalent Value |
|--------------------|--------------------|
| `1e3` | 1000 |
| `9e9` | 9,000,000,000 |
| `3e4` | 30,000 |
| `2e9` | 2,000,000,000 |

### Why PHP Accepts Scientific Notation

PHP's `is_numeric()` function considers scientific notation strings to be valid numeric values. This is by design — PHP is a loosely typed language that tries to be flexible with number formats.

### Why `strlen()` Counts Characters, Not Value

`strlen()` counts the number of **characters in the string**, not the numeric value. So:

| Input | `strlen()` result | Numeric Value | Passes all checks? |
|-------|------------------|---------------|--------------------|
| `1000` | 4 — too long | 1000 | ❌ |
| `999` | 3 — OK | 999 | ❌ not > 999 |
| `9e9` | 3 — OK | 9,000,000,000 | ✅ |
| `3e4` | 3 — OK | 30,000 | ✅ |
| `2e9` | 3 — OK | 2,000,000,000 | ✅ |
| `1e3` | 3 — OK | 1000 | ✅ |

`9e9` is only 3 characters long — it passes `strlen() < 4`. PHP evaluates it as 9,000,000,000 — it passes `> 999`. And `is_numeric("9e9")` returns `true` — it passes the numeric check. All three conditions are satisfied simultaneously.

---

## 5. Crafting the Payload — Scientific Notation Bypass

Any of the following payloads satisfy all three conditions:

```
9e9
3e4
2e9
1e3
5e5
```

The payload is submitted in the password field. The URL becomes:

```
https://[challenge-url]/?password=9e9
```

<p align="center">
<img width="824" height="313" alt="image" src="https://github.com/user-attachments/assets/203d83dd-4305-491e-9fc8-234d23792c19" />
</p>

---

## 6. Flag Obtained

The server processes the input — all three checks pass — and the flag is revealed:

<p align="center">
<img width="824" height="313" alt="image" src="https://github.com/user-attachments/assets/64a81a15-a6cc-4418-bba4-3d09f4c97cf9" />
</p>

> **FLAG: obtained from the server response**

---

# OWASP Classification

> **OWASP Top 10: A03 — Injection**
> The application used PHP's `is_numeric()` and a loose numeric comparison (`>`) to validate input. PHP's type juggling caused `is_numeric("9e9")` to return `true` and `"9e9" > 999` to evaluate as `true` — because PHP automatically converts the scientific notation string to its float value during comparison. The `strlen()` check was bypassed because it measures string length, not numeric value. This is a logic flaw in the validation implementation caused by PHP's loosely typed comparison behaviour.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| `is_numeric()` accepts scientific notation | Input `9e9` is 3 chars long but equals 9 billion |
| `strlen()` checks string length not numeric value | `9e9` is only 3 characters — bypasses the length restriction |
| Loose PHP comparison `>` evaluates type-coerced values | PHP converts `"9e9"` to float `9000000000.0` before comparing |
| All three conditions satisfied simultaneously | Flag revealed with a 3-character scientific notation input |

### The Correct Fix

```php
// VULNERABLE — is_numeric() accepts scientific notation
// strlen() checks character count not numeric value
if (is_numeric($password) && strlen($password) < 4 && $password > 999)

// SECURE — cast to integer first, then check value range
$password = intval($_GET['password']);
if ($password > 999 && $password < 9999) {
    // valid range — but now 1000-9999 are all 4 chars anyway
}

// BETTER FIX — reject scientific notation explicitly
if (is_numeric($password) && !preg_match('/[eE]/', $password) && strlen($password) < 4 && $password > 999)

// BEST FIX — use strict integer validation
if (ctype_digit($password) && intval($password) > 999)
```

The root cause is relying on `is_numeric()` for security validation. This function accepts floats, negative numbers, and scientific notation — making it unsuitable as a strict input validator. Using `ctype_digit()` instead ensures only plain digit strings are accepted, rejecting `9e9` entirely.

### PHP Type Juggling Reference

| Function | Accepts `9e9`? | Why |
|----------|---------------|-----|
| `is_numeric("9e9")` | ✅ Yes | PHP treats scientific notation as numeric |
| `strlen("9e9")` | Returns `3` | Counts characters, not numeric value |
| `"9e9" > 999` | ✅ True | PHP converts `"9e9"` to float before comparing |
| `ctype_digit("9e9")` | ❌ No | Only accepts plain digit characters `0-9` |
| `intval("9e9")` | Returns `9` | Only reads up to the `e` — not 9 billion |

---

# Exploitation Flow (OWASP A03 Mapping)

1. Attacker accesses the challenge page — password prompt asking for a number that can exceed 999
2. **"wanna source?"** link clicked — full PHP validation logic revealed
3. Source code analysed — three conditions identified: `is_numeric`, `strlen < 4`, `> 999`
4. Contradiction identified — no plain integer satisfies all three simultaneously
5. PHP's `is_numeric()` behaviour researched — scientific notation strings are accepted
6. `strlen("9e9")` evaluated — only 3 characters long, passes the length check
7. `"9e9" > 999` evaluated — PHP converts to float 9,000,000,000 — passes the value check
8. Payload crafted: `9e9` — satisfies all three conditions simultaneously
9. Submitted via URL: `?password=9e9`
10. Server executes `die($flag)` — flag revealed

---

# Conclusion

This challenge demonstrates **PHP type juggling** — a class of vulnerability caused by PHP's loosely typed nature. When PHP compares a string like `"9e9"` to an integer using `>`, it automatically converts the string to its float equivalent `9000000000.0` before comparing. This means a 3-character string can represent a 10-digit number — bypassing a `strlen()` check that was intended to prevent large numbers.

The challenge is a reminder that `is_numeric()` is not a strict validator — it accepts scientific notation, floats, and signed numbers. In security-sensitive contexts, `ctype_digit()` should be used instead.

To prevent this type of bypass, applications should:

- **Never use `is_numeric()` for strict integer validation** — use `ctype_digit()` instead
- **Cast input to integer with `intval()` before comparing** — `intval("9e9")` returns `9`, not 9 billion
- **Validate both the format and the value** — string length checks are not a substitute for numeric range checks
- **Be aware of PHP type juggling** — loose comparisons (`==`, `>`, `<`) perform automatic type conversion that can produce unexpected results
