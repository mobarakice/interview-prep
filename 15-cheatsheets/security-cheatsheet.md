# Security Reference Cheatsheet

> Quick-reference checklist for application security, OAuth2, cryptographic algorithms, and web threat mitigation

---

## 1. Core Threat Mitigations

### Cross-Site Scripting (XSS)
- **Checklist**:
  - Set `HttpOnly` and `Secure` flags on authorization cookies.
  - Escape all dynamic user output on frontend views.
  - Apply CSP headers: `Content-Security-Policy: default-src 'self'`.

### Cross-Site Request Forgery (CSRF)
- **Checklist**:
  - Apply `SameSite=Strict` cookie policies.
  - Enforce Double Submit Cookie anti-CSRF token verification on all POST/PUT/DELETE API gateways.

### SQL Injection (SQLi)
- **Checklist**:
  - Always use prepared statements and parameterized queries (default in Hibernate / Spring Data).
  - Validate and sanitize input bounds for dynamic query fields.

---

## 2. Cryptographic Selection Guide

| Action | Recommended | Avoid | Rationale |
|---|---|---|---|
| **Data Encryption** | AES-256-GCM | AES-CBC | GCM provides AEAD (Authenticated Encryption with Associated Data). |
| **Password Hashing** | Argon2id, BCrypt | SHA-256, MD5 | Plain hashing is vulnerable to fast GPU rainbow-table cracking. |
| **Asymmetric Keys** | RSA-4096, ECDSA | RSA-1024 | RSA-1024 is mathematically weak and breakable. |
| **Transport Layer** | TLS 1.3 | TLS 1.0, SSLv3 | Old protocols contain known vulnerabilities (POODLE, BEAST). |

---

## 3. OAuth 2.0 Auth Flow Reference

- **PKCE (Proof Key for Code Exchange)**: Mandatory for Public Clients (SPAs, Mobile). Uses code challenge and verifier to prevent interception.
- **Client Credentials Flow**: For machine-to-machine integrations (Microservice A calling Microservice B). No user context involved.
- **Authorization Code Flow**: For traditional web server applications. Returns code on frontend, exchanges code for tokens on backend.
