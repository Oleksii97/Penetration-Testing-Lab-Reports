
 Penetration Testing Vulnerability Report

**Date:** May 23, 2024  
**Status:** Critical  
**Target:** Web Application (Admin Panel)

### 1. Vulnerability Summary
*   **Title:** Privilege Escalation via Cookie Manipulation.
*   **Vulnerability Type:** OWASP A01:2021 – Broken Access Control.
*   **Severity:** Critical.
*   **CVSS v3.1 Vector:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` (**Score: 9.8**)

### 2. Technical Description
The vulnerability exists due to the application's reliance on client-side authorization data stored in plaintext. The backend fails to validate the `user_role` cookie against the session record in the database. Consequently, any user can modify their local cookie parameters within the browser, resulting in a complete authentication bypass.

### 3. Proof of Concept (PoC)
1.  **Authentication:** Log in to the application as a standard visitor.
2.  **Developer Tools:** Press `F12` (or `Ctrl+Shift+I`) to open the Browser Developer Tools.
3.  **Data Manipulation:**
    *   Navigate to the **Application** tab (Chrome/Edge) or **Storage** tab (Firefox).
    *   In the left sidebar, expand **Cookies** and select the application's domain.
    *   Locate the cookie named `user_role`.
    *   Double-click the value `visitor` and change it to `admin`.
4.  **Escalation:** Refresh the page (F5) and press  "send" after we can see "Validation password"


```

### 4. Business Impact
This vulnerability allows an unauthorized user to gain Superuser privileges. Impacts include:
*   Exfiltration of all users' Personally Identifiable Information (PII).
*   Unauthorized deletion or modification of critical business data.
*   Complete compromise of the confidentiality, integrity, and availability (CIA triad) of the system.

### 5. Remediation
1.  **Server-side Authorization:** User roles must be determined strictly on the server side based on a secure session ID mapped in the database.
2.  **Secure Cookie Attributes:** Implement `HttpOnly` and `Secure` flags for cookies. More importantly, do not store authorization logic/roles in plaintext on the client side.
3.  **Signed Tokens (JWT):** If a stateless approach is required, use JSON Web Tokens (JWT) signed with a strong cryptographic algorithm (e.g., RS256) to prevent unauthorized tampering.
