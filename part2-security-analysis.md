# Part 2 – Security Analysis  
## Chosen Risk: Broken Access Control / Cross-Tenant IDOR (Multi-Tenant Isolation Failure)

### 1) Technical explanation (how the attack works)

**Scenario:** A multi-tenant CRM where requests include identifiers such as `company_id`, `customer_id`, `invoice_id`, etc. The UI shows only the user’s tenant data, but the backend mistakenly trusts client-supplied identifiers.

**Step-by-step attack flow:**

1. **Attacker creates a normal account**  
   They sign up as a legitimate user for Tenant A (their own company workspace).

2. **Attacker identifies an object ID used in requests**  
   Using browser dev tools (Network tab) or a proxy (e.g., Burp), they observe requests like:  
   - `GET /dashboard/customers?company_id=123`  
   - `GET /api/customers/98765`  
   - `GET /api/invoices/555001?company_id=123`

3. **Attacker modifies the identifier (IDOR attempt)**  
   They change `company_id=123` to another value (e.g., `company_id=124`) or change an object ID (e.g., `/api/customers/98766`) and replay the request.

4. **Backend fails object-level authorization**  
   The server checks “is the user logged in?” but **does not check** “does this user belong to Tenant B / own this object?”  
   If the request succeeds, the attacker now receives records for another tenant.

5. **Attacker iterates/enumerates IDs**  
   If IDs are predictable (sequential integers) or discoverable, the attacker can repeat the attack across many objects (customers, deals, invoices, notes), producing a widening data leak.

6. **Exfiltration and persistence**  
   The attacker exports data (via endpoints or repeated calls). If the system logs are weak, they can remain undetected longer—especially if they access data at a “low and slow” rate.

---

### 2) Real-world example (similar breach/incident)

A relevant example is **Peloton’s 2021 API exposure**, where researchers reported that Peloton’s backend APIs could return user data even for profiles set to private due to authorization weaknesses in certain endpoints—an example of the same “authenticated but not authorised for this object” pattern seen in IDOR/BOLA-style failures. :contentReference[oaicite:0]{index=0}

---

### 3) Defense strategy (3 security controls)

#### Control 1: Centralised, server-side object-level authorisation (deny-by-default)

**What to implement:**  
- Enforce object-level checks on every read/write action (UI and API).  
- Use a consistent policy approach (e.g., service-layer authorization middleware, or a policy engine like OPA/Casbin).  
- Pattern: `authorize(user, action, resource)` before any data access.  
- Ensure the check validates both:
  - **tenant scope** (same company/tenant)
  - **role/permission** (e.g., Admin vs standard user)

**Why it works:**  
It prevents trusting client-supplied IDs. Even if an attacker changes `customer_id` or `company_id`, the request is rejected unless the resource belongs to the caller’s tenant and role permissions allow access.

**How to verify it works:**  
- Add automated tests for BOLA/IDOR:
  - “User from Tenant A cannot access Tenant B resources” across all endpoints.
- Create negative tests for common endpoints (customers, invoices, notes, exports).
- Run DAST tests that mutate IDs and confirm responses are consistently **403** (or equivalent).

**Trade-off:**  
Higher engineering effort and maintenance. A central policy model is powerful, but it requires disciplined implementation and may slow feature development if teams are not used to building “secure by default.”

---

#### Control 2: Tenant isolation at the data layer (e.g., Postgres Row-Level Security or enforced tenant filters)

**What to implement:**  
- Enforce tenant boundaries in the database layer:
  - Postgres **Row-Level Security (RLS)** with tenant-aware policies, *or*
  - strict query patterns requiring `tenant_id` filtering in every query, ideally enforced by a shared data access layer.
- Ensure DB roles are least-privileged and cannot bypass tenant filters.

**Why it works:**  
It reduces blast radius even if an application-layer check is missed. If a request attempts to fetch an object from another tenant, the database refuses to return rows outside the tenant boundary.

**How to verify it works:**  
- Unit/integration tests that attempt cross-tenant reads/writes at the data access layer.
- Security test: deliberately disable/bypass an app-layer check in a staging environment and confirm RLS still blocks cross-tenant access.
- Audit: confirm DB users/roles cannot disable RLS or query unrestricted tables.

**Trade-off:**  
Increased complexity in schema design, migrations, and debugging. RLS and strict tenant filtering can also introduce performance considerations and require careful indexing and query tuning.

---

#### Control 3: Detection + throttling for suspicious access patterns (rate limits, anomaly alerts, audit logs)

**What to implement:**  
- API rate limiting and request throttling per user/IP/token.  
- Structured audit logging for sensitive actions:
  - object access events (who accessed what, tenant_id, endpoint, outcome)
  - exports and bulk reads
- Alerts for anomalies:
  - repeated 403s (probing)
  - rapid sequential object access (enumeration)
  - unusual export volume or off-hours spikes

**Why it works:**  
Even strong prevention controls can fail due to human error. Detection and throttling reduce the speed and scale of exploitation and increase the chance of early response.

**How to verify it works:**  
- Simulate ID enumeration attempts and confirm:
  - rate limits trigger
  - alerts fire
  - logs contain the required fields for investigation (user, tenant, object, timestamp, IP)
- Run tabletop exercises: confirm SOC/on-call can trace “what was accessed” within minutes.

**Trade-off:**  
Potential UX impact (false positives) and ongoing operational cost. Monitoring, alert tuning, and log storage can be expensive and require continuous maintenance to avoid alert fatigue.

---

### 4) Summary of why this risk matters

Multi-tenant systems depend on tenant isolation as a core trust guarantee. IDOR/BOLA-style failures are often easy to exploit, high impact, and frequently missed when authorization is implemented inconsistently across UI and API layers. A strong strategy combines **consistent authorization**, **data-layer isolation**, and **monitoring/response readiness**. :contentReference[oaicite:1]{index=1}
