# Join Team - CyberTalents

---

## Challenge Description
**Flag safe in the server environment, can you reveal it.**

## Challenge Link
https://cybertalents.com/challenges/web/join-team

---

## 1. Initial Interface

The challenge is accessed via the CyberTalents platform, which displays the challenge title, description, and a link to launch the target web application.

<p align="center">
<img width="686" height="380" alt="image" src="https://github.com/user-attachments/assets/f3f9978c-037d-439e-ba77-e2472131dac3" />
</p>

---

## 2. Challenge Page — Home Page

Upon navigating to the challenge URL, a company homepage is presented titled **"Our Great Great Great Company"**. The page contains standard company content with no obvious interactive functionality or visible attack surface at first glance.

<p align="center">
<img width="947" height="400" alt="image" src="https://github.com/user-attachments/assets/09071737-4e90-4ee2-b2ca-715fb7d0fdd3" />
</p>

---

## 3. About Us Page

Navigating to the **About Us** page reveals generic company information. There is nothing interactive or exploitable on this page. The investigation continues to other pages.

<p align="center">
<img width="944" height="374" alt="image" src="https://github.com/user-attachments/assets/d9f771f2-b85a-4b7f-9c91-a82ff0626e4e" />
</p>

---

## 4. Work For Us Page — Upload Functionality Discovered

Navigating to the **Work For Us** page reveals a **CV upload form**. The page contains a file input field and displays the following message:

> **"Only PDF files are allowed, don't try to bypass the validation we have strong security ( or we think so :) )"**

<p align="center">
<img width="947" height="387" alt="image" src="https://github.com/user-attachments/assets/076a5ac9-5b26-4118-8113-e0fd45dd4a72" />
</p>

The hint **"or we think so :)"** is an intentional signal from the challenge designer — it tells us the validation is weak and likely bypassable. This upload functionality immediately becomes the primary target.

### What is Unrestricted File Upload?

**Unrestricted File Upload** is a critical web vulnerability that occurs when a server accepts uploaded files without properly validating their actual type, content, or extension. If an attacker can upload a file that the server will later **execute** — such as a PHP script — they can achieve **Remote Code Execution (RCE)**, meaning they can run any command on the server.

| Validation Type | How It Works | Can It Be Bypassed? |
|----------------|--------------|-------------------|
| **Client-side only** | JavaScript checks file type in the browser | ✅ Yes — easily disabled |
| **Extension check** | Server checks if filename ends in `.pdf` | ✅ Yes — rename the file |
| **Content-Type check** | Server checks the `Content-Type` header | ✅ Yes — modify in Burp Suite |
| **Magic bytes check** | Server reads the actual file content | Harder but still possible |

In this challenge, the server only checks the **filename extension** and **Content-Type header** — both of which are fully user-controlled and can be manipulated using Burp Suite.

---

## 5. Testing File Upload — Plain Text File Rejected

A plain text file `test.txt` is uploaded to test the server's validation. The server correctly rejects it with:

> **"Only PDF document allowed"**

<p align="center">
<img width="249" height="57" alt="image" src="https://github.com/user-attachments/assets/1ab47a96-8ab7-479e-8b23-dd0542b411b0" />
</p>

This confirms the server is performing some level of file type checking.

---

## 6. Uploading a Legitimate PDF — Success

A legitimate PDF file `test.pdf` is uploaded. The server accepts it and responds with:

> **"Your CV has been uploaded successfully in test.pdf"**

<p align="center">
<img width="323" height="52" alt="image" src="https://github.com/user-attachments/assets/73f8d4f5-df09-41d8-a2b9-294c5a497b32" />
</p>

A clickable link `data/test.pdf` is provided which opens the uploaded file directly in the browser. This reveals two critical pieces of information:

- The upload directory is **`data/`**
- Uploaded files are **directly accessible via URL** — meaning if a PHP file were uploaded there, the server would **execute it** when accessed

---

## 7. Creating a PHP Test File

A PHP test file `test.php` is created containing a simple PHP command to confirm whether the server executes PHP files:

```php
<?php
    echo 'test'
?>
```

<p align="center">
<img width="595" height="200" alt="image" src="https://github.com/user-attachments/assets/da515b27-8a28-4bce-a5b0-7457e8d5a026" />
</p>

This file is then uploaded through the browser to check if the server accepts it without manipulation.

---

## 8. Intercepting the Upload Request in Burp Suite

The upload of `test.php` is intercepted using **Burp Suite Proxy**. The captured POST request shows exactly how the browser sends the file to the server:

```
POST /jobs HTTP/1.1
Host: [challenge-url]
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="test.php"
Content-Type: application/x-php

<?php echo 'test'; ?>
------WebKitFormBoundary--
```

<p align="center">
<img width="701" height="338" alt="image" src="https://github.com/user-attachments/assets/0fb9a83f-394d-48a3-af58-eb201d988662" />
</p>

Two fields in this request control the file type validation:

| Field | Current Value | What the Server Checks |
|-------|--------------|----------------------|
| `filename=` | `test.php` | File extension — server sees `.php` |
| `Content-Type:` | `application/x-php` | MIME type — server sees it is not a PDF |

---

## 9. Sending Request to Repeater — Initial Response

The intercepted POST request is forwarded to **Burp Suite Repeater** for further analysis. The initial response confirms the server rejected the file:

> **"Only PDF document allowed"**

<p align="center">
<img width="716" height="398" alt="image" src="https://github.com/user-attachments/assets/24771490-f7c8-420e-a614-3f31f06c54b0" />
</p>

This tells us the server is checking either the filename extension, the Content-Type header, or both — and both of these can be manipulated directly in Burp Suite Repeater.

---

## 10. Bypassing the Validation — Content-Type Manipulation

In Burp Suite Repeater, two values in the request are modified to disguise the PHP file as a PDF:

**Before:**
```
filename="test.php"
Content-Type: application/x-php
```

**After:**
```
filename="test.pdf"
Content-Type: application/pdf
```

<p align="center">
<img width="717" height="360" alt="image" src="https://github.com/user-attachments/assets/ac116308-5d16-46a0-81fb-ca43b0ea81e4" />
</p>

The modified request is resent and the server responds with:

> **"Your CV has been uploaded successfully in test.pdf"**

<p align="center">
<img width="370" height="66" alt="image" src="https://github.com/user-attachments/assets/9a2a6f7d-e6b1-4aff-b6a1-fa87598de7d6" />
</p>

---

## 11. Accessing the Uploaded File — Bypass Confirmed

The URL is modified by replacing `/jobs` with `/data/test.pdf` to access the uploaded file directly:

```
http://[challenge-url]data/test.pdf
```

<p align="center">
<img width="695" height="54" alt="image" src="https://github.com/user-attachments/assets/f577f279-bdec-4fb7-bece-be6e50ba2453" />
</p>

<p align="center">
<img width="717" height="371" alt="image" src="https://github.com/user-attachments/assets/f811952b-ae1d-44ad-8dc3-bee1253b2f6b" />
</p>

The server executes the PHP file and returns the output — confirming:

- The server **accepted a PHP file disguised as a PDF**
- The file is **stored and accessible** in the `data/` directory
- The server **executes PHP files** when accessed via URL

This means the application is fully vulnerable to **Remote Code Execution via unrestricted file upload**.

### Why Is This Vulnerable?

The server only checks the `Content-Type` header and filename extension sent by the **client** in the HTTP request — both of which are completely user-controlled. The server never inspects the **actual file content** to verify it is really a PDF. This means:

- A PHP file renamed to `.pdf` with `Content-Type: application/pdf` passes all server checks
- Once uploaded, the file sits in the publicly accessible `data/` directory
- Accessing it via URL causes the server to **execute the PHP code inside it**
- An attacker can exploit this to run any command on the server

---

## 12. What is Weevely?

**Weevely** is an open-source web shell tool used in penetration testing and CTF challenges. It is pre-installed on **Kali Linux** and allows a security researcher to establish a remote terminal-like connection to a web server through an uploaded PHP backdoor file.

### How Weevely Works

Weevely has two components:

| Component | Description |
|-----------|-------------|
| **The backdoor PHP file** | An obfuscated PHP script generated by Weevely that listens for incoming commands |
| **The Weevely terminal client** | A command-line tool on Kali Linux that sends encrypted commands to the backdoor |

### What Does the Generated Backdoor Contain?

When Weevely generates a backdoor PHP file, it produces a **heavily obfuscated PHP script** that is password-protected. Under the obfuscation, it:

1. Listens for incoming HTTP requests containing encoded commands from the Weevely client
2. Decodes those commands using the shared password
3. Executes the commands on the server using PHP's `eval()` function
4. Returns the output of the commands back to the attacker's terminal

The file looks like random scrambled PHP code to a human reader — this is intentional obfuscation to make it harder for security scanners to detect.

### How to Install Weevely

Weevely comes pre-installed on Kali Linux. To verify or install it:

```bash
# Check if already installed
weevely

# Install if not present
sudo apt install weevely
```

### Generating the Backdoor

```bash
weevely generate [your-kali-password] /home/kali/Desktop/backdoor.php
```

> **Note:** Use a less suspicious filename than `backdoor.php` when uploading — something like `cv_upload.php` blends in better and is less likely to raise flags.

---

## 13. Uploading the Backdoor via Burp Suite

The generated `backdoor.php` file is uploaded through the CV upload form. The same Burp Suite bypass from Step 10 is applied:

- Change `filename="backdoor.php"` → `filename="backdoor.pdf"`
- Change `Content-Type: application/x-php` → `Content-Type: application/pdf`

<p align="center">
<img width="712" height="375" alt="image" src="https://github.com/user-attachments/assets/13904352-7b43-4c23-92eb-fa9ed9132ebb" />
</p>

The server responds with:

> **"Your CV has been uploaded successfully in backdoor.pdf"**

---

## 14. Activating the Backdoor & Connecting via Weevely

The backdoor is activated by accessing it via the `data/` directory URL:

```
http://[challenge-url]data/backdoor.pdf
```

Back in the Kali Linux terminal, Weevely connects to the backdoor:

```bash
weevely http://[challenge-url]data/backdoor.pdf [your-kali-password]
```

<p align="center">
<img width="743" height="82" alt="image" src="https://github.com/user-attachments/assets/ad4ebc02-359b-472e-8638-f43775dc1865" />
</p>

A remote shell session is now established on the server.

---

## 15. Extracting the Flag

With the remote shell active, the following commands are run on the server:

```bash
ls
cat index.php
```

<p align="center">
<img width="578" height="183" alt="image" src="https://github.com/user-attachments/assets/d5626e48-103e-4d8b-acf7-34f2326b6bfd" />
</p>

The flag is located on the second line of `index.php`:

> **FLAG: `{Your_Flag_Here}`**

> **Note:** If a communication error occurs, regenerate a new backdoor with a different filename, re-upload it using the same bypass method, and reconnect using the new URL.

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A04 — Insecure Design / Unrestricted File Upload**

The server fails to properly validate uploaded files — it only checks the client-supplied `Content-Type` header and filename extension, both of which are fully user-controlled. This allows an attacker to upload and execute arbitrary PHP code on the server, achieving full Remote Code Execution.

---

# Why This Is a Security Issue

The application's file upload validation is entirely dependent on **client-supplied values** that can be changed by anyone:

- Changing `Content-Type` to `application/pdf` is enough to bypass the check
- The server never inspects the actual file content to verify it is a real PDF
- Uploaded PHP files are stored in a **publicly accessible directory** (`data/`)
- Accessing the uploaded file via URL causes the server to **execute it**
- This allows full **Remote Code Execution** — the attacker can run any command on the server

---

# Exploitation Flow (OWASP A04 Mapping)

1. Attacker explores the website — home and about pages show nothing exploitable
2. **Work For Us** page reveals a CV upload form
3. The hint "or we think so :)" signals weak validation
4. A `.txt` file is rejected — confirming some validation exists
5. A legitimate `.pdf` is accepted — upload directory `data/` is discovered
6. A PHP file is uploaded directly — rejected by the server
7. Burp Suite intercepts the upload POST request
8. `filename` changed from `.php` to `.pdf` and `Content-Type` changed to `application/pdf`
9. Modified request forwarded — server accepts the disguised PHP file
10. File accessed via `data/[filename]` — PHP executes, bypass fully confirmed
11. Weevely generates an obfuscated PHP backdoor file on Kali Linux
12. Backdoor uploaded using the same Content-Type bypass technique
13. Backdoor activated by accessing it via the `data/` URL
14. Weevely connects and establishes a remote shell on the server
15. `cat index.php` executed — **flag extracted**

---

# Conclusion

This challenge demonstrates **Unrestricted File Upload** leading to **Remote Code Execution**. The server's validation was entirely dependent on client-supplied values — the filename extension and Content-Type header — which are trivially manipulated using Burp Suite. Once a PHP backdoor was uploaded and activated, full server access was achieved using Weevely.

To prevent unrestricted file upload vulnerabilities, applications should:

- **Never trust client-supplied headers** like `Content-Type` for security decisions
- **Validate file content** using magic bytes — check the actual file structure, not just the name
- **Store uploaded files outside the web root** so they cannot be executed via URL
- **Rename uploaded files** server-side to random strings — never keep the original filename
- **Disable PHP execution** in the upload directory using `.htaccess`
- Use an **allowlist** for permitted file types — only accept exactly what is needed
