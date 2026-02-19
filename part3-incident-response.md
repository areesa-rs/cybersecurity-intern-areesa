# Part 3 – Incident Response

## Scenario Overview

A support agent reported that they were able to access another company’s customer records by modifying a URL parameter:

/customers/view?company_id=1234 → changed to company_id=1235

The request succeeded and returned another tenant’s data.  
This indicates a cross-tenant authorization failure (IDOR / broken access control).

---

# 1) Immediate Actions (First 60 Minutes)

The priority is containment, preservation of evidence, and impact assessment.

### Step 1 – Escalate and Declare Security Incident (0–10 minutes)
- Notify Security Lead / Engineering Manager.
- Open an incident ticket with severity level (likely High/Critical).
- Freeze deployments affecting authentication/authorization until assessed.

### Step 2 – Contain the Vulnerability (10–25 minutes)
- Immediately disable or restrict the affected endpoint.
  - Temporary mitigation options:
    - Remove direct object lookup by `company_id`
    - Add emergency tenant-check validation
    - Restrict access to internal-only while investigating
- If needed, enable a feature flag to disable customer-view functionality temporarily.

### Step 3 – Preserve Logs and Evidence (25–40 minutes)
- Secure:
  - Web server logs
  - API access logs
  - Database query logs
  - Audit logs (if available)
- Ensure logs are retained and protected from rotation/deletion.
- Capture timestamp of discovery.

### Step 4 – Initial Exposure Assessment (40–60 minutes)
- Identify:
  - Which endpoints are affected
  - Whether the issue applies only to `/customers/view`
  - Whether similar patterns exist in other endpoints
- Determine preliminary scope: Is this limited to read access or does write access also bypass controls?

The goal within the first hour is to **stop further exposure** and preserve forensic evidence.

---

# 2) Investigation Checklist

## A) How Widespread Is This Vulnerability?

- Does the issue affect:
  - Only `/customers/view`?
  - All endpoints that use `company_id`?
  - API endpoints as well as the web dashboard?
- Are other objects vulnerable?
  - Invoices
  - Orders
  - Notes
  - Attachments
- Is the vulnerability limited to GET requests, or does it apply to:
  - PUT (update)
  - DELETE
  - Export/download endpoints?
- Is authorization enforced consistently across services?

---

## B) What Data Was Potentially Exposed?

- Which fields are returned in the response?
  - PII (names, emails, phone numbers)
  - Financial data (invoices, payment status)
  - Internal notes
- Were attachments accessible?
- Was export functionality available?
- What tenant(s) could be accessed from which accounts?

---

## C) Was It Exploited Before This Report?

- Review logs for:
  - Repeated sequential `company_id` access attempts
  - Access to company IDs not belonging to the user
  - High-volume object access patterns
  - Repeated 200 responses for cross-tenant IDs
- Look for:
  - Enumeration patterns (1234 → 1235 → 1236)
  - Unusual time-of-day access
  - Large response sizes
- Check:
  - Export logs
  - Bulk download activity
  - Unusual admin account behavior

If logs are insufficient, note this as a logging control gap.

---

# 3) Root Cause Analysis

### Possible Technical Cause 1: Missing Object-Level Authorization Check
The system validates that the user is authenticated but does not verify:
- The requested `company_id` belongs to the user’s tenant.

This is a classic IDOR / Broken Object-Level Authorization flaw.

---

### Possible Technical Cause 2: Client-Side Enforcement Only
The UI hides other tenants’ data, but server-side validation was never implemented.  
The backend trusted the `company_id` parameter from the request.

---

### Possible Technical Cause 3: Inconsistent Authorization Logic
Some endpoints use proper authorization middleware, while others bypass it due to:
- Legacy code
- Direct database queries
- Microservice inconsistencies

---

### Most Likely Cause

The most likely cause is **missing object-level authorization checks at the server side**, because:

- The attack required only parameter manipulation.
- The request returned data successfully (no error handling triggered).
- This pattern strongly matches a backend that checks “is logged in” but not “is allowed to access this tenant’s data.”

---

# 4) Fix Validation

When a developer says “I fixed it,” verification must be independent and systematic.

## Test 1 – Manual Cross-Tenant Manipulation
- Log in as Tenant A user.
- Attempt:
  /customers/view?company_id=anotherTenantID
- Expected result:
  - 403 Forbidden (or equivalent)
  - No data leakage in response body.

---

## Test 2 – Object Enumeration Test
- Attempt sequential company_id changes.
- Confirm:
  - All cross-tenant requests are denied.
  - No data is returned.
  - No error messages reveal sensitive information.

---

## Test 3 – API-Level Testing
- Repeat manipulation at API level (not just UI).
- Test:
  - GET
  - PUT
  - DELETE
  - Export endpoints
- Ensure consistent authorization enforcement.

---

## Test 4 – Role Variation Testing
Test across:
- Standard user
- Support agent
- Admin
- Super-admin (if exists)

Confirm tenant boundaries are enforced regardless of role (unless explicitly designed otherwise).

---

## Test 5 – Automated Regression Test

- Add automated test case:
  - “User from Tenant A cannot access Tenant B resource.”
- Ensure CI pipeline fails if authorization regression occurs.

---

## Test 6 – Log Verification

- Confirm:
  - Unauthorized attempts are logged.
  - Logs contain user ID, tenant ID, endpoint, timestamp.
  - Alerts trigger for repeated failed access attempts.

---

# Conclusion

This incident represents a high-severity tenant isolation failure.  
Immediate containment, forensic log preservation, and comprehensive authorization review are essential.

Verification must go beyond “it works in the UI” and include:

- API-level testing
- Role-based testing
- Enumeration attempts
- Automated regression safeguards

In multi-tenant SaaS systems, tenant isolation is a non-negotiable security boundary. Any cross-tenant access must be treated as a critical incident.
