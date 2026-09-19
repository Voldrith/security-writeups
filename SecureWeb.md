# SecureWeb — Building a Deliberately Vulnerable Web Application

**Context:** Project built during penetration testing internship at Vieh Group
**Purpose:** Demonstrate, document, and teach common web application attack vectors in a controlled environment

---

## Why I Built It

During my internship, I wanted a safe, repeatable environment to practice and demonstrate web application vulnerabilities end-to-end — from discovery through exploitation and remediation.

Rather than rely on third-party lab platforms, I built **SecureWeb**: a deliberately vulnerable web application covering the OWASP Top 10 in a realistic app structure.

The goal wasn't just to find vulnerabilities — it was to document them the way a real pentester does: with reproduction steps, impact analysis, and remediation guidance.

---

## Vulnerabilities Implemented

| Category | Details |
|---|---|
| SQL Injection | Classic auth bypass + UNION-based extraction |
| Cross-Site Scripting (XSS) | Stored and reflected variants |
| IDOR / Broken Access Control | Direct object reference on user records |
| Authentication Bypass | Weak session handling, predictable tokens |
| Command Injection | Unsanitised input passed to system calls |
| Insecure File Upload | Unrestricted file type leading to RCE |
| Business Logic Flaws | Price tampering, workflow bypass |

Each vulnerability was implemented the way it appears in the wild — not artificially stubbed — so exploitation felt realistic.

---

## Example — SQL Injection to Auth Bypass

**Vulnerable endpoint:**

    POST /login
    username=admin' OR '1'='1'--&password=anything

**Result:** Authenticated as admin without valid credentials.

**Root cause:** Login query concatenated user input directly:

    SELECT * FROM users WHERE username = '$user' AND password = '$pass';

**Impact:** Full authentication bypass — attacker gains admin access with no valid credentials.

**Remediation:**
- Use parameterised queries / prepared statements
- Never concatenate user input into SQL
- Apply least privilege to the database user
- Add rate limiting and logging on auth endpoints

---

## What I Learned

- **Building vulnerabilities teaches you how to find them.** Writing the vulnerable code made me understand why the vulnerability exists — not just how to exploit it.
- **Reporting is half the job.** A finding without reproduction steps, evidence, and remediation is just a note. Structured reporting is what clients actually pay for.
- **Realistic beats artificial.** Stubbing a vuln is easy. Making it feel like a real misconfiguration is what makes the training useful.

---

## Result

SecureWeb is now used as a documentation and demo reference for common web attack vectors, with writeups structured the same way as production penetration test findings.
