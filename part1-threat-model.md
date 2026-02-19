# Part 1 – Threat Model

## 1. Critical Attack Surfaces

### 1.1 Web Dashboard (Multi-Tenant Access Control / IDOR)

**Threat (What could go wrong):**  
A user manipulates object identifiers (e.g., `company_id` or `customer_id`) within requests to access records belonging to another tenant. This represents broken access control or an Insecure Direct Object Reference (IDOR), where the application fails to properly enforce tenant isolation at the server side.

**Impact:**  
Cross-tenant exposure of personally identifiable information (PII), contracts, invoices, and internal notes. This would trigger breach notification requirements, damage enterprise trust, and potentially lead to regulatory penalties.

**Likelihood:**  
High – Broken access control is consistently ranked among the most common web vulnerabilities, especially in multi-tenant SaaS environments where access checks may be inconsistently implemented.

---

### 1.2 API Backend (Authorization Gaps / Object-Level Access Control)

**Threat (What could go wrong):**  
The API verifies that a user is authenticated but fails to verify whether the user is authorised to access a specific object. An attacker could script API calls to enumerate or scrape large volumes of data across tenants.

**Impact:**  
Large-scale data extraction that may not be immediately visible in the UI. Automated scraping increases the scale and speed of compromise, making detection and containment more difficult.

**Likelihood:**  
High – APIs often contain more endpoints than are documented or fully tested. Object-level authorization checks are frequently overlooked in backend services.

---

### 1.3 Authentication (Credential Stuffing / Weak MFA / Token Mismanagement)

**Threat (What could go wrong):**  
An attacker uses leaked credentials from previous breaches to perform credential stuffing attacks. If multi-factor authentication (MFA) is weak or not enforced, administrative accounts may be compromised. Poor token handling (e.g., long-lived tokens or lack of rotation) may further increase risk.

**Impact:**  
Full account takeover, potentially including administrator accounts. This would allow data export, customer communication abuse, privilege escalation, and persistent access.

**Likelihood:**  
Medium–High – Most SaaS platforms are continuously targeted with credential stuffing attempts. The likelihood depends heavily on rate limiting, MFA enforcement, and monitoring controls.

---

### 1.4 S3 Storage (Misconfiguration / Insecure Access Patterns)

**Threat (What could go wrong):**  
Cloud storage buckets may be misconfigured with overly permissive access controls (e.g., public object ACLs), or object keys may be predictable. Sensitive attachments could be accessed directly without authentication.

**Impact:**  
Exposure of sensitive documents such as identity documents, invoices, and internal records. This bypasses application-level protections and may remain undetected for extended periods.

**Likelihood:**  
Medium – While less common than in previous years, misconfigured cloud storage still occurs, particularly in rapidly deployed or poorly reviewed environments.

---

### 1.5 Database Layer (Injection / Over-Privileged Roles)

**Threat (What could go wrong):**  
Improper input validation or unsafe query construction may allow SQL injection. Additionally, the application may use an over-privileged database role, increasing the blast radius of compromise.

**Impact:**  
Direct database read/write access, potential modification or deletion of data, and worst-case full tenant database extraction.

**Likelihood:**  
Medium – Modern ORMs reduce risk but do not eliminate it. Custom filters, reporting queries, or legacy endpoints often reintroduce injection risks.

---

## 2. Top 3 Prioritised Risks

### Risk 1: Broken Access Control / Cross-Tenant IDOR

This represents the highest risk because tenant isolation is fundamental in enterprise SaaS systems. A single flaw could expose multiple companies’ data, leading to immediate reputational and regulatory consequences.

---

### Risk 2: API Authorization Flaws Enabling Bulk Extraction

API vulnerabilities scale significantly due to automation. Even a small authorization gap can allow scripted bulk extraction of sensitive records, amplifying impact beyond isolated incidents.

---

### Risk 3: Account Takeover (Credential Stuffing + Weak MFA)

Account compromise is both common and highly damaging. Once an attacker gains access to an administrative account, they can export data, escalate privileges, and maintain persistence within the system.

---

## 3. Prioritisation Logic

The prioritisation is based on a combination of impact and likelihood, with particular emphasis on tenant isolation and scalability of abuse.

Cross-tenant data exposure was ranked highest because enterprise onboarding depends on strict separation between customers. Any violation of this boundary represents a systemic failure with severe legal and reputational consequences.

API authorization flaws were prioritised due to their ability to scale through automation. Unlike manual UI exploitation, API abuse enables rapid and large-scale data extraction.

Account takeover was included because credential-based attacks are extremely common in SaaS environments. When combined with weak MFA controls, they directly lead to data exfiltration and administrative abuse.

Overall, the top three risks represent high-impact scenarios that are either common in real-world SaaS environments or capable of scaling rapidly once exploited.
