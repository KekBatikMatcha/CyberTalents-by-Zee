# Admin Has The Power - CyberTalents

---

## Challenge Description
**Administrators only has the power to see the flag, can you be one?**

## Challenge Link
https://cybertalents.com/challenges/web/admin-has-the-power

---

## 1. Initial Interface

The challenge introduces a web application where only administrators are allowed to access the flag.

This is the starting point of the application.

<p align="center">
<img width="900" height="368" alt="image" src="https://github.com/user-attachments/assets/877e16af-492d-4c53-b87d-e43fb04a1d30" />
</p>

---

## 2. Login Interface

The application presents a login form requiring a username and password.

No valid credentials are directly provided at this stage, indicating that they may be hidden within the application.

<p align="center">
<img width="959" height="389" alt="image" src="https://github.com/user-attachments/assets/b72aed92-82ab-410f-ae95-8d6d8d8dc0e0" />
</p>

---

## 3. Source Code Analysis

By inspecting the page source code, a hidden comment is discovered containing credentials:

> "For maintenance purposes, use this info (username = support and password = x34245323)"

This reveals hardcoded credentials exposed on the client side.

<p align="center">
<img width="959" height="385" alt="image" src="https://github.com/user-attachments/assets/15e011f1-50ec-4cc4-b586-c2b7a5d76e43" />
</p>

---

## 4. Successful Login

Using the discovered credentials (`support`), authentication is successful and access to the user dashboard is granted.

<p align="center">
<img width="959" height="394" alt="image" src="https://github.com/user-attachments/assets/e21e7641-b656-4879-ab67-6ca33ae65111" />
</p>

---

## 5. Cookie Inspection (Browser DevTools)

Using browser Developer Tools (Application tab), the session cookie is inspected.

A `role` parameter is identified, which is responsible for controlling user privileges.

<p align="center">
<img width="959" height="387" alt="image" src="https://github.com/user-attachments/assets/4abab43f-3ec2-4425-807a-2ccc9f8e4c60" />
</p>

---

## 6. Cookie Manipulation

The role value is modified from:

`support` → `admin`


After refreshing the page, the application grants elevated privileges.

<p align="center">
<img width="400" height="29" alt="image" src="https://github.com/user-attachments/assets/579d9082-77ae-4d5a-8dfa-2c5f67b0478c" />
</p>

---

## 7. Flag Obtained

After modifying the cookie, admin access is granted and the flag is revealed:

> Admin Secret flag: `hiadminyouhavethepower`

<p align="center">
<img width="937" height="355" alt="image" src="https://github.com/user-attachments/assets/8087a3b6-98dd-44f9-80da-bba8e6c0566a" />
</p>

---

# Burp Suite Analysis

---

## 8. Burp Suite Setup

Burp Suite is launched with Proxy Intercept enabled to capture HTTP traffic between the browser and the server.

<p align="center">
<img width="953" height="462" alt="image" src="https://github.com/user-attachments/assets/19f06923-1cae-4825-b610-730149e03586" />
</p>

---

## 9. Accessing the Challenge via Burp Browser

The challenge URL is opened using the Burp Suite embedded browser to enable traffic interception.

<p align="center">
<img width="959" height="491" alt="image" src="https://github.com/user-attachments/assets/cd6a58e1-dc48-4ad4-bb2c-a1e92aba79da" />
</p>

---

## 10. Intercepting Login Request

The login request is intercepted while using the `support` credentials, capturing session-related data.

<p align="center">
<img width="269" height="107" alt="image" src="https://github.com/user-attachments/assets/adf86d96-99d1-48de-8bff-c0412120457c" />
</p>

---

## 11. Sending Request to Repeater

The intercepted request is sent to the Burp Suite Repeater tool for further analysis and modification.

<p align="center">
<img width="959" height="413" alt="image" src="https://github.com/user-attachments/assets/a4ab311c-60e9-4d30-83ff-206ffb9ffb8b" />
</p>

---

## 12. Response Analysis

The request is processed in Repeater, allowing inspection of the server’s response behavior.

<p align="center">
<img width="221" height="314" alt="image" src="https://github.com/user-attachments/assets/c948f10c-c673-45c6-8e01-174cf728f465" />
</p>

---

## 13. Cookie Manipulation via Burp Suite

The session cookie is identified in the request and modified:

role=`support` → role=`admin`


The modified request is then resent to the server.

<p align="center">
<img width="302" height="42" alt="image" src="https://github.com/user-attachments/assets/6207a278-85cb-41d7-808e-59c33db756ab" />
</p>

---

## 14. Final Exploitation Result

After resending the modified request, the server grants admin access and reveals the flag:

> Admin Secret flag: `hiadminyouhavethepower`

<p align="center">
<img width="727" height="347" alt="image" src="https://github.com/user-attachments/assets/3eb20dda-8b84-4e21-9c98-693ca930fb23" />
</p>

---

# OWASP Classification

This vulnerability falls under:

> **OWASP Top 10: A01 - Broken Access Control**

Broken Access Control occurs when an application fails to properly enforce user permissions, allowing users to perform actions or access data beyond their intended role.

---

# Why this is a security issue

The application stores user roles in a client-side cookie and trusts this value without proper validation.

This allows attackers to:
- Modify their role from user → admin  
- Bypass authorization controls  
- Gain unauthorized access to restricted functionality  

---

# Exploitation Flow (OWASP A01 Mapping)

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

# Conclusion

This challenge demonstrates **OWASP A01: Broken Access Control**, where insecure handling of client-side cookies allows privilege escalation due to missing server-side authorization checks.
