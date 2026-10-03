[README.md](https://github.com/user-attachments/files/33001232/README.md)
# Mediroza Hospital — Patient Portal Penetration Test

> **Training / lab engagement.** `medirozahospital.com`, "Mediroza General Hospital," and all patient, staff, and shareholder records referenced below are fictional data generated for a security-testing lab exercise. Nothing in this repository documents or targets a real organization.

A black-box web application penetration test against a hospital patient portal, broken into three milestones (**M1 → M3**) plus a consolidated client-facing report (**M4**). This repo contains the walkthrough, evidence screenshots, and the final report.

---

## Contents

- [Engagement Summary](#engagement-summary)
- [Findings Summary](#findings-summary)
- [M1 — Initial Access](#m1--initial-access)
- [M2 — Data Extraction](#m2--data-extraction)
- [M3 — Attack (Cracking): Staff Salaries & Shareholder Data](#m3--attack-cracking-staff-salaries--shareholder-data)
- [M4 — Penetration Testing Report](#m4--penetration-testing-report)
- [Recommendations](#recommendations)
- [Disclaimer](#disclaimer)

---

## Engagement Summary

| | |
|---|---|
| **Client** | Mediroza General Hospital (fictional, lab environment) |
| **Target** | `medirozahospital.com` (199.188.201.16) |
| **Engagement type** | Black-box web application penetration test |
| **Platform** | Kali Linux |
| **Tester** | Muhammad |

**Objective:** determine whether an external, unauthenticated attacker could compromise the confidentiality of patient health records and other sensitive hospital data, by (1) gaining initial access to the patient portal, (2) defeating the encryption protecting any data retrieved, and (3) locating further sensitive business data beyond the original scope.

## Findings Summary

| ID | Finding | Severity |
|----|---------|----------|
| F-1 | SQL Injection — Authentication Bypass (Patient Portal) | 🔴 Critical |
| F-2 | Sensitive Backup File Exposed via Directory Listing (`/old/`) | 🔴 Critical |
| F-3 | Weak / Predictable PDF Encryption Passwords on Lab Reports | 🟠 High |
| F-4 | Username Enumeration on Login Form | 🟡 Medium |
| F-5 | Verbose Database Error Messages | 🟢 Low |

---

## M1 — Initial Access

**Task:** attack the website and retrieve 3 confidential patient PDF lab reports.

### Reconnaissance

Passive recon (WHOIS, DNS) was run first to confirm ownership and the public IP of the target, followed by a WhatWeb scan to fingerprint the stack.

| | |
|---|---|
whois-lookup
<img width="1365" height="479" alt="01-whois-lookup" src="https://github.com/user-attachments/assets/18eaf35d-9555-4461-92dd-6730227a8769" />

whois-lookup
<img width="1365" height="573" alt="02-whois-registrant-details" src="https://github.com/user-attachments/assets/dc00ef72-a485-473c-9b8e-7278a72bd55b" />


| *WHOIS lookup of medirozahospital.com* | *WHOIS registrant details* |

<img width="1362" height="392" alt="03-whois-nameservers" src="https://github.com/user-attachments/assets/7d1dc1ce-3065-4736-9aef-c186eb5112f3" />

<img width="1365" height="141" alt="04-dns-resolution-nslookup" src="https://github.com/user-attachments/assets/a7726f24-e623-4afb-a8f4-a9afef541fa9" />


| *WHOIS record — name servers* | *DNS resolution confirming the public IP* |

The app is served by LiteSpeed behind a Cloudflare-style JS challenge on direct HTTP access.

| | |
|---|---|
whatweb-fingerprint
<img width="1349" height="153" alt="05-whatweb-fingerprint" src="https://github.com/user-attachments/assets/782d5838-d5ae-4c49-b3bd-95efd6a38c04" />

robots-txt-antibot-page
<img width="1365" height="553" alt="06-robots-txt-antibot-page" src="https://github.com/user-attachments/assets/b3cf1970-ba59-49ec-a563-d234ba5cf021" />

| *WhatWeb technology fingerprint* | *robots.txt returning an anti-bot interstitial* |

<img width="1361" height="564" alt="07-antibot-js-challenge-1" src="https://github.com/user-attachments/assets/53f5aadd-9a23-4e7f-999f-8d8390cedac3" />

<img width="1346" height="517" alt="08-antibot-js-challenge-2" src="https://github.com/user-attachments/assets/752341f5-dca9-4ee6-93e4-08669ea3f9b3" />

| *Anti-bot challenge markup* | *Anti-bot challenge logic* |

### Finding F-4: Username Enumeration

The `/patient/login.php` form returned an explicit **"Username not found"** error for invalid usernames, distinct from the response for a valid username with a wrong password — enabling username enumeration.

<img width="1358" height="627" alt="09-patient-portal-login" src="https://github.com/user-attachments/assets/31e39328-38a8-43eb-abba-71f18b3fd014" />

*Patient Portal login form*

<img width="1365" height="586" alt="10-username-enumeration" src="https://github.com/user-attachments/assets/507766ba-f5e1-438e-b568-93a356270877" />

*Explicit "Username not found" error*

### Finding F-1: SQL Injection — Authentication Bypass

A single quote in the username field triggered a raw MySQL syntax error, confirming unsanitized input was concatenated directly into a SQL query.

<img width="1357" height="514" alt="11-sql-injection-error" src="https://github.com/user-attachments/assets/11f28268-3603-4189-b411-a5b65edc50d8" />

*Raw MySQL error confirming SQL injection*

An `admin' --`-style payload then bypassed authentication entirely:

<img width="1365" height="508" alt="12-sql-injection-auth-bypass" src="https://github.com/user-attachments/assets/c5ff20c1-542c-40df-b3bd-04c4e0b3c41b" />

*Authentication-bypass payload*

This granted access to the portal's "My lab reports" page, listing three encrypted patient PDFs:

<img width="1360" height="587" alt="13-patient-portal-access-granted" src="https://github.com/user-attachments/assets/4cfa77f5-05cf-4e13-8cf4-7c6b361686bd" />

*Authenticated access to 3 encrypted lab reports (LR-2024-1187, -1192, -1205)*

> **Impact — Critical.** Full authentication bypass via SQLi lets any unauthenticated remote attacker access any patient's protected health records.

---

## M2 — Data Extraction

**Task:** crack the encryption on all 3 retrieved files.

The three PDFs were downloaded locally for offline analysis; each was password-protected.

| | |
|---|---|

| *PDF password prompt* | *The 3 downloaded patient report PDFs* |

<img width="1365" height="576" alt="14-pdf-password-prompt" src="https://github.com/user-attachments/assets/8726f6cc-a221-44f0-adc5-0c5d67d054c9" />

downloaded-encrypted-pdfs

<img width="1042" height="357" alt="15-downloaded-encrypted-pdfs" src="https://github.com/user-attachments/assets/9e6d4155-b13b-4d14-ba50-35ed22d49213" />

### Finding F-3: Weak / Predictable PDF Passwords

A crackable hash (`pdf2john` / hashcat-compatible, PDF R3, 128-bit) was extracted from each file and run through a basic dictionary attack.

<img width="931" height="752" alt="16-pdf-hash-extraction" src="https://github.com/user-attachments/assets/664a7353-0d43-43a2-a4e7-91648c2a13ad" />

*Password hash extracted from patient_report_1.pdf*

**Report 1 (S. Dlamini)** — cracked on the first attempt: `123456`

<img width="902" height="687" alt="17-password-crack-report1-123456" src="https://github.com/user-attachments/assets/5e6ed8ab-f43e-4e16-a8c5-89f91230f278" />

<img width="962" height="637" alt="18-decrypted-report-dlamini" src="https://github.com/user-attachments/assets/6f01b893-ca4c-4a3a-aa98-dcdc5a98788c" />

*Decrypted pathology report — Sipho Dlamini (LR-2024-1187)*

**Report 2 (P. Reddy)** — cracked on the second attempt: `password`

<img width="907" height="862" alt="19-password-crack-report2-password" src="https://github.com/user-attachments/assets/1ad5554c-e67f-4763-93a6-ee531171cb1f" />

<img width="857" height="595" alt="20-decrypted-report-reddy" src="https://github.com/user-attachments/assets/a13bf0c7-b824-405b-b7b3-3834e921554b" />

*Decrypted pathology report — Priya Reddy (LR-2024-1192)*

**Report 3 (E. Thompson)** — cracked the same way:

<img width="912" height="600" alt="21-decrypted-report-thompson" src="https://github.com/user-attachments/assets/362777e2-894d-473e-b302-b058036e38ae" />

*Decrypted pathology report — Emily Thompson (LR-2024-1205)*

> **Impact — High.** The files were nominally encrypted, but common dictionary passwords defeated that protection in seconds.

---

## M3 — Attack (Cracking): Staff Salaries & Shareholder Data

**Task:** find staff salaries and shareholder details of the hospital.

### Finding F-2: Sensitive Backup File Exposed via Directory Listing

Further enumeration found an unlinked `/old/` directory with directory listing enabled, exposing a full SQL database backup with **no authentication required**.

<img width="1586" height="445" alt="22-exposed-backup-directory" src="https://github.com/user-attachments/assets/fc37b90c-b40f-4be6-85b8-d77a38f8a0c3" />

*`Index of /old/` exposing `mediroza_db_backup_2019.sql`*

The backup contained data far outside the original "patient lab reports" scope:

*Staff salary register recovered from the backup*

<img width="842" height="712" alt="23-staff-salary-register" src="https://github.com/user-attachments/assets/4f02c05c-275b-4bf4-8b3a-0b6e54f1b906" />

*Shareholder register and equity percentages*

<img width="687" height="322" alt="24-shareholder-register" src="https://github.com/user-attachments/assets/ba4f93b6-36e4-444e-89dd-205142c51b6a" />

> **Impact — Critical.** A publicly downloadable, unauthenticated database backup containing payroll and shareholder data is a severe breach of staff and corporate confidentiality — and requires no skill to exploit, just browsing to the path.

---

## M4 — Penetration Testing Report

M4 is the consolidated, client-facing write-up built from the M1–M3 evidence above: executive summary, scope, methodology, full findings table, and recommendations.

<img width="817" height="266" alt="image" src="https://github.com/user-attachments/assets/6e570f99-b4b1-4eae-ad29-ed3082568234" />

<img width="832" height="296" alt="image" src="https://github.com/user-attachments/assets/856a15a0-91da-438b-8ac6-475c944fe6b9" />

---

## Recommendations

| Finding | Key remediation |
|---|---|
| **F-1** SQL Injection | Use parameterized queries everywhere; never concatenate user input into SQL. Add a WAF as defense-in-depth. |
| **F-2** Exposed backup | Remove `/old/` from the web root immediately; disable directory listing; store backups outside the document root, encrypted at rest. |
| **F-3** Weak PDF passwords | Use strong, unique, randomly generated passwords/keys per document, or move to authenticated in-portal delivery instead of relying on PDF passwords. |
| **F-4** Username enumeration | Return a generic "invalid username or password" message; add rate limiting / lockout. |
| **F-5** Verbose errors | Disable verbose DB error output in production; log server-side only. |

---

## Disclaimer

This content was produced for a controlled, authorized security-training lab using synthetic data. It is shared for educational purposes (OWASP-style web app testing methodology, reporting structure) and must not be used against systems you do not own or have explicit written authorization to test.
