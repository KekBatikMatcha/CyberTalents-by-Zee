# Admin Has The Power - CyberTalents

## Challenge Description
**Administrators only has the power to see the flag, can you be one?**

## Challenge Link
http://cdcamxwl32pue3e6m5p6v4ehxzg1rm236mnkmugv0-web.cybertalentslabs.com/

## Initial Interface

This is the initial CyberTalents challenge page for **"Admin Has The Power"**.

It introduces the objective of the challenge, where only administrators are allowed to access the flag.

<p align="center">
<img width="959" height="368" alt="image" src="https://github.com/user-attachments/assets/877e16af-492d-4c53-b87d-e43fb04a1d30" />
</p>

##  2. Login Interface

The login page displays a username and password field.

However, no valid credentials are provided directly to the user at this stage, suggesting that they may be hidden elsewhere.

<p align="center">
<img width="959" height="389" alt="image" src="https://github.com/user-attachments/assets/b72aed92-82ab-410f-ae95-8d6d8d8dc0e0" />
</p>

---

##  3. Source Code Analysis

By inspecting the page source code, a hidden comment is discovered:

> "For maintenance purposes, use this info (username = support and password = x34245323)"

This indicates hardcoded credentials exposed in the client-side source code.

<p align="center">
<img width="959" height="385" alt="image" src="https://github.com/user-attachments/assets/15e011f1-50ec-4cc4-b586-c2b7a5d76e43" />
</p>

##  4. Successful Login

Using the discovered credentials (`support`), successful authentication is achieved and access to the user dashboard is granted.

<p align="center">
<img width="959" height="394" alt="image" src="https://github.com/user-attachments/assets/e21e7641-b656-4879-ab67-6ca33ae65111" />
</p>

---

##  5. Cookie Inspection (Browser DevTools)

Using browser Developer Tools → Application tab, the session cookie is inspected.

A `role` value is identified, which controls user privileges.

<p align="center">
<img width="959" height="387" alt="image" src="https://github.com/user-attachments/assets/4abab43f-3ec2-4425-807a-2ccc9f8e4c60" />
</p>

##  6. Cookie Manipulation

The role value is modified from `support` to `àdmin`

After refreshing the page, access privileges are escalated.

<p align="center">
<img width="959" height="29" alt="image" src="https://github.com/user-attachments/assets/579d9082-77ae-4d5a-8dfa-2c5f67b0478c" />
</p>


##  7. Flag Obtained

After modifying the cookie, the application grants admin access and reveals the flag:

> Admin Secret flag: `hiadminyouhavethepower`

<p align="center">
<img width="937" height="355" alt="image" src="https://github.com/user-attachments/assets/8087a3b6-98dd-44f9-80da-bba8e6c0566a" />
</p>

---

##  Burp Suite Analysis

## 🛠️ 8. Burp Suite Setup

Burp Suite is launched and Proxy Intercept mode is enabled.  
The browser is opened through Burp Suite to capture HTTP traffic.

<p align="center">
<img width="953" height="462" alt="image" src="https://github.com/user-attachments/assets/19f06923-1cae-4825-b610-730149e03586" />
</p>

## 🌐 9. Accessing Challenge via Burp Browser

The challenge URL is opened inside the Burp Suite embedded browser to allow interception of requests.

<p align="center">
<img width="959" height="491" alt="image" src="https://github.com/user-attachments/assets/cd6a58e1-dc48-4ad4-bb2c-a1e92aba79da" />
</p>

## 🔐 10. Intercepting Login Request

During login using the `support` credentials, Burp Suite captures the HTTP request containing session cookies.

<p align="center">
<img width="269" height="107" alt="image" src="https://github.com/user-attachments/assets/adf86d96-99d1-48de-8bff-c0412120457c" />
</p>

## 📤 11. Sending Request to Repeater

The intercepted request is forwarded to the **Repeater** tool for modification and testing.

<p align="center">
<img width="959" height="413" alt="image" src="https://github.com/user-attachments/assets/a4ab311c-60e9-4d30-83ff-206ffb9ffb8b" />
</p>

## 🔁 12. Response Analysis

The server response is analyzed after sending the request, revealing the current session behavior.

<p align="center">
<img width="221" height="314" alt="image" src="https://github.com/user-attachments/assets/c948f10c-c673-45c6-8e01-174cf728f465" />
</p>

## ⚔️ 13. Cookie Manipulation via Burp Suite

The `role=support` value in the request header is modified to: `admin`

The request is resent.

<p align="center">
<img width="937" height="430" alt="image" src="https://github.com/user-attachments/assets/2bc26e2c-80e0-4c67-8a2e-ae6f2313447f" />
</p>

<p align="center">
<img width="302" height="42" alt="image" src="https://github.com/user-attachments/assets/6207a278-85cb-41d7-808e-59c33db756ab" />
</p>

## 🏁 14. Final Exploitation Result

After modifying the cookie in Burp Suite and resending the request, the server returns admin access and reveals the flag:

> Admin Secret flag: `hiadminyouhavethepower`

<p align="center">
<img width="727" height="347" alt="image" src="https://github.com/user-attachments/assets/3eb20dda-8b84-4e21-9c98-693ca930fb23" />
</p>

---

## 🔐 OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A01 - Broken Access Control**

Broken Access Control occurs when an application fails to properly enforce user permissions, allowing users to perform actions or access data beyond their intended role.

---

## ❗ Why this is a security issue

In this application, user roles are stored in a client-side cookie.

The server incorrectly trusts this value without validation.

This allows attackers to:
- Modify their role from user → admin
- Bypass authorization controls
- Gain unauthorized access to restricted functionality

---

## ⚔️ Exploitation Flow (OWASP A01 Mapping)

<p align="center">
<img width="377" height="291" alt="image" src="https://github.com/user-attachments/assets/74e37b94-a862-4fea-bd8c-d90edb106d45" />
</p>

1. User logs in successfully  
2. Server assigns role (user)  
3. Role is stored in a cookie  
4. Application uses cookie for authorization decisions  
5. Attacker modifies cookie value (user → admin)  
6. Server fails to validate authenticity of role  
7. Access control is bypassed  

---

## 🏁 Conclusion

This challenge demonstrates **OWASP A01: Broken Access Control**, where user roles stored in client-side cookies can be manipulated to escalate privileges due to missing server-side authorization checks.















