Description: **Administrators only has the power to see the flag , can you be one ?**
Link: **http://cdcamxwl32pue3e6m5p6v4ehxzg1rm236mnkmugv0-web.cybertalentslabs.com/**

<img width="959" height="368" alt="image" src="https://github.com/user-attachments/assets/877e16af-492d-4c53-b87d-e43fb04a1d30" />

<img width="959" height="389" alt="image" src="https://github.com/user-attachments/assets/b72aed92-82ab-410f-ae95-8d6d8d8dc0e0" />

<img width="959" height="385" alt="image" src="https://github.com/user-attachments/assets/15e011f1-50ec-4cc4-b586-c2b7a5d76e43" />

<img width="959" height="394" alt="image" src="https://github.com/user-attachments/assets/e21e7641-b656-4879-ab67-6ca33ae65111" />

<img width="959" height="387" alt="image" src="https://github.com/user-attachments/assets/4abab43f-3ec2-4425-807a-2ccc9f8e4c60" />

<img width="959" height="29" alt="image" src="https://github.com/user-attachments/assets/579d9082-77ae-4d5a-8dfa-2c5f67b0478c" />

<img width="937" height="355" alt="image" src="https://github.com/user-attachments/assets/8087a3b6-98dd-44f9-80da-bba8e6c0566a" />

**Burp suite**

<img width="953" height="462" alt="image" src="https://github.com/user-attachments/assets/19f06923-1cae-4825-b610-730149e03586" />

<img width="959" height="491" alt="image" src="https://github.com/user-attachments/assets/cd6a58e1-dc48-4ad4-bb2c-a1e92aba79da" />

<img width="269" height="107" alt="image" src="https://github.com/user-attachments/assets/adf86d96-99d1-48de-8bff-c0412120457c" />

<img width="959" height="413" alt="image" src="https://github.com/user-attachments/assets/a4ab311c-60e9-4d30-83ff-206ffb9ffb8b" />

<img width="221" height="314" alt="image" src="https://github.com/user-attachments/assets/c948f10c-c673-45c6-8e01-174cf728f465" />

<img width="937" height="430" alt="image" src="https://github.com/user-attachments/assets/2bc26e2c-80e0-4c67-8a2e-ae6f2313447f" />

<img width="302" height="42" alt="image" src="https://github.com/user-attachments/assets/6207a278-85cb-41d7-808e-59c33db756ab" />

<img width="727" height="347" alt="image" src="https://github.com/user-attachments/assets/3eb20dda-8b84-4e21-9c98-693ca930fb23" />

## 🔐 OWASP Classification

This vulnerability falls under **OWASP Top 10: A01 - Broken Access Control**.

Broken Access Control occurs when an application fails to properly enforce user permissions, allowing users to perform actions or access data beyond their intended role.

## ❗ Why this is a security issue

In this application, user roles are stored in a client-side cookie.

The server incorrectly trusts this value without validation.

This allows attackers to:
- Modify their role from user → admin
- Bypass authorization controls
- Gain unauthorized access to restricted functionality

## ⚔️ Exploitation Mapping to OWASP A01

<img width="377" height="291" alt="image" src="https://github.com/user-attachments/assets/74e37b94-a862-4fea-bd8c-d90edb106d45" />


1. User logs in successfully
2. Server assigns role (user)
3. Role is stored in a cookie
4. Application uses cookie for authorization decisions
5. Attacker modifies cookie value (user → admin)
6. Server fails to validate authenticity of role
7. Access control is bypassed

This challenge demonstrates OWASP A01: Broken Access Control, where user roles stored in client-side cookies can be manipulated to escalate privileges due to missing server-side authorization checks.

















