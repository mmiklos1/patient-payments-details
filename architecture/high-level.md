# Patient Payments — High-Level Architecture (Dental PMS)

Basic architecture for a public-facing online patient payment system.

---

## 1. UI (Angular — latest version)

### 1.1 Landing & practice identity

- Each **dental practice** gets a **unique hash** that represents their landing page (practice-based).
- If different **locations** within a practice cannot share billing info, new API endpoints will be needed for cross-location communication.

### 1.2 ROI widget & location analytics

- An ROI widget will correlate **online payments** to **returned payments**.
- For **location-level** (vs practice-level) analytics, the patient payments table must support:
  - `payment_id`, `patient_id`, `location_id`
  - A **subtable or JSON column** linking individual billing codes to payments.

### 1.3 Post-login experience

- After login, the patient is redirected to **view their bill** and choose how much to pay.

**Open question:** Show an **itemized bill** or only a **sum total?**  
**Recommendation:** Because partial payments typically apply to the oldest charges first, allowing only a **partial payment** (no per-line selection) is usually sufficient and avoids unnecessary complexity.

### 1.4 General UI workflow

| Step | Action |
|------|--------|
| 1 | User goes to landing page (from bill/QR). |
| 2 | User logs in with **account number** and **last name** (head-of-household last name). |
| 3 | *(See open question below.)* |
| 4 | Payment page shows **itemized charges**: ordered by date, separated by patient; use **procedure names**, not procedure codes. |
| 5 | Patient chooses **full** or **partial** amount only (no per-line payment). Partial payments apply **oldest to newest**. |
| 6 | Patient enters payment info (same page vs next page depends on payment portal; same page preferred even if tight on mobile). |
| 7 | Payment provider handles processing/security; on success, API hook redirects user to **Receipt** page. |
| 8 | Receipt shows what was paid and when, broken down like the itemized charges (for HSA/dispute reasons). |
| 9–11 | Receipt header/body/download/email — see below. |

**Open question:** How are **head-of-household/guarantors** and patient bills separated in the database? Billing is to the head-of-household; all bills must be easily tied to them.

**Partial payments & receipt:** Apply the payment to individual billing items (oldest first) before building the receipt. For ties (e.g. same day, same appointment), use a consistent rule (e.g. youngest-to-oldest or DB-efficient order).

### 1.5 Receipt requirements

**Header**

- Date of payment  
- Transaction ID  
- Payment method (e.g. VISA ****1234)  
- Total amount paid today  

**Body**

- Itemized breakdown per procedure (grouped by patient). MVP can simplify, but the data model should support this since allocation is needed for the receipt anyway.  
- Signature area (practice and/or PMS logo).  

**Actions**

- **Download** receipt (PDF preferred).  
- **Email** receipt to a chosen address.  
- **Print** — prints a pdf-like version of the receipt page (browser print); user can save as PDF. No API/Redis; use when Redis data has expired (e.g. user active on page >30 min) or as fallback if download fails.

**Note:** Receipt data in Redis ~30 min TTL for email/PDF. JWT expires after 10 min idleness (redirect out). If user stays on receipt page >30 min, download/email generation no longer available; **Print** remains. Receipt URL may include transaction ID for same-session revisit. After the session ends, re-visiting the receipt is not required; support and practice/tenant UIs should be able to look up payment and receipt details from the database. **Online patient payment** must be a distinct payment type so practice/tenant views and the ROI widget can show “paid online.”

---

## 2. API — New endpoints and JWT

### 2.1 Patient login (JWT)

- Patient has a bill with an **account number** (per head-of-household; add if missing).
- Access: scan QR on bill or open website; login with **account number + last name**.
- JWT stored in **HttpOnly** cookie/header; backend-only. No DB persistence required for this login.

**Note:** A full patient portal can be added later; the DB will already support viewing payments.

### 2.2 New endpoints

| Endpoint / flow | Purpose |
|-----------------|--------|
| **PaymentCompleted hook** | Called by payment solution (or UI) when a payment is taken. Builds itemized receipt data, redirects user to receipt page. Receipt payload can be cached in Redis ~30 min TTL for PDF/email. |
| **Download Receipt PDF** | Server-side PDF generation and download (authoritative; UI could be manipulated). |
| **Email Receipt** | Send receipt via email. Options below. |

### 2.3 Email receipt options

- **No attachment (API Gateway + SES):** Direct API Gateway → SES with template; JSON maps to to/from/body. Can link back to transaction instead of attaching PDF; no PDF attachment.
- **Third-party mailer (e.g. Mailhog, Mailchimp):** UI calls API; API builds and sends via 3rd party with configured “sender.” Implementation depends on chosen provider.
- **With PDF attachment:** UI → API → **Lambda** (build MIME + attach PDF) → SES. Offload heavy work to Lambda to avoid blocking the API and long user wait.

### 2.4 Security — separate Payments API

- This system is **public-facing**. Use a **dedicated Payments API** that:
  - Has **no direct DB access**.
  - Talks to the **main EMR API** via API key or (preferred) **AWS security group** restricting access to the Payments API’s internal IP.

### 2.5 EMR endpoints

- Because we don't have a database connection in the Payments API, we'll need endpoints added to the EMR as well. Anything that pulls from the DB will need its own endpoint:
  - Get HoH charges
  - POST transactions once PaymentCompleted is called
  - Get receipt data

---

## 3. Database — New tables and table updates

*(Assumes existing EMR schema; changes should be minimal.)*

1. **Payment source**  
   Where “how payment was taken” is stored (table or column in payments/transactions): add value **ONLINE PATIENT PAYMENT**.

2. **Payment/transaction table**  
   Ensure a column (or equivalent) for the **payment provider reference ID** for reconciliation and support.

3. **Payment application (allocation)**  
   If missing, add a **payment_application** (or equivalent) concept: links **transaction/payment IDs** to **charge/procedure IDs**. Can be a separate table for performance or folded into procedure/charge table.

4. **Active Practice Hash Table**
   Purpose: Maps URL landing page to practice demographics, and allow patient searches under the correct practic/company for quicker lookup. Must contain **hash**, **tenantId**, **companyID (if applicable)**, hash is a unique identifier and should be ~6 alphanumeric characters not including the handful that can be hard to read and are typically excluded from hashes.

---

## 4. Metrics and analytics

- **Request timing (histograms):**  
  Measure duration for: login, receipt build, email send, payment finalization, and every UI request/sub-request. Surfaces outliers and supports debugging. Export so Datadog/Prometheus can consume.

- **Funnel / conversion:**  
  Per-practice (and optionally per-location): landing → login → payment completed. Use for marketing, ROI, and debugging. Metrics must be consumable by Datadog or Prometheus.

- **Error / failure tracking:**  
  Track and log failure cases (e.g. Get HoH charges failed after retry) so they are visible in metrics and logs for debugging and support. No PII/PCI in logs.

---

## 5. Product Questions

- **Synchronicity**  
  Assume the payment provider takes the payment successfully, but our side (or theirs) fails to receive the "payment completed" hook. Our database will not have the payment updated, but the patient will have a charge on their account. In this scenario, the practice should still have the payment on their balance sheets/banking statement, even if our side doesn't reflect that. Do we want a reconciliation job to pull all transactions from the payment solution and ensure we have all their reference IDs stored (e.g. nightly or weekly)?

  A cleaner approach: do not expose the hook from the API directly; instead expose an endpoint from the Gateway that connects to SQS, which then connects to the API. Once the API successfully handles the request and updates the DB, the item is removed from the queue. If it fails to be handled after a number of times, it moves to a DLQ so we can investigate. This guarantees we always process a payment unless the payment provider themselves failed (and they already have fallbacks in place).

- **Rollbacks**  
  The main webhook that handles completed payments would also receive a rollback from the provider (e.g. when it goes through the bank/card). The webhook will need to be able to POST the chargeback to the transactions table through the API. Do we have a ROLLBACK (or equivalent) status on the transactions table to indicate this? This would go through the same SQS workflow described above if that approach is used.

- **Disputes**  
  For disputes, the money is removed from the practice while the dispute is in place. The webhook will receive this; charges should be updated to "Disputed" (or the PMS equivalent). Since the money is gone while the charge is in dispute, we'd typically show the patient that they owe those charges again. Does the PMS have a way to indicate open disputes? Is there any desire to show a charge as in dispute in the patient payment UI, given we don't allow payment by line item?

---

## 6. Completeness analysis

| Section | Completeness | Notes |
|--------|--------------|--------|
| **UI** | Strong | Rate limiting and error states are covered in the flowchart. Accessibility is a given. |
| **API** | Good | EMR endpoints added in brief; majority of contract in place. Idempotency not required (payment handled by 3rd party). Reconciliation (e.g. hook failure after provider success) out of scope for this doc. |
| **Metrics** | Good | **No PII/PCI in logs.** Logging for error handling will be added to track failures. |

Use this table to plug remaining gaps before or while building the flowchart.

---

## TL;DR — Build checklist (repos, services, infrastructure)

Physical things to create before or while building. Not code-level tasks; use this to spin up repos, services, and infra.

---

### 1. Repositories

| Create | Purpose |
|--------|---------|
| **Patient Payments API repo** | Backend for the public-facing payments flow (no direct DB; calls EMR API). |
| **Patient Payments UI repo** | Angular SPA: landing, make-payment, receipt. |

---

### 2. Patient Payments API (new service)

| Create | Purpose |
|--------|---------|
| **GET** credentials / login | Validate account number + last name; issue JWT (HttpOnly). |
| **GET** itemized charges | Fetch HoH charges for payment page (calls EMR). |
| **Webhook** (PaymentCompleted) | Receives payment completion from provider; build receipt, POST transactions to EMR, cache receipt in Redis. |
| **GET** receipt | Return receipt data (e.g. for PDF/email; from cache or EMR). |
| **GET** download receipt PDF | Server-side PDF generation and download. |
| **POST** email receipt | Send receipt via email (SES, 3rd-party mailer, or Lambda+SES). |

---

### 3. Redis (instance + usage)

| Create | Purpose / behavior |
|--------|---------------------|
| **Redis instance** | Used by Payments API (and optionally EMR if landing data is cached there). |
| **Landing page demographics** | Cache practice/details by hash. TTL ~1 day — landing page details won’t update for ~1 day after a practice updates their details. |
| **Patient charge details / receipt payload** | Same cache key: itemized charges for the session, updated once payment is received into receipt data. TTL ~30 min for PDF/email generation (and when DB save fails, user can still get receipt). |

---

### 4. EMR / PMS API (new or updated endpoints)

| Create | Purpose |
|--------|---------|
| **GET** credentials / login | Validate patient login (account + last name); used by Payments API. |
| **GET** itemized charges (HoH) | Return charges for head of household; used by Payments API. |
| **POST** transactions (and update charges) | Record payment and apply to charges; called by Payments API after webhook. May already exist; ensure it supports online patient payment source and provider reference ID. |
| **GET** receipt data | Return receipt details for a transaction; used for PDF/email when needed from EMR. |

---

### 5. UI — Angular app (pages / entrypoints)

| Create | Purpose |
|--------|---------|
| **Landing page** | Practice hash in URL; practice details, phone number; login (account number + last name). |
| **Make a payment page** | Itemized charges, amount (full/partial), payment info (inline or separate step per provider). |
| **Receipt page** | Header, body, actions: view, download PDF, email, print. |

---

### 6. Database (EMR / PMS schema)

| Create | Purpose |
|--------|---------|
| **Practice / patient payments hash table** | Maps landing-page hash to practice (e.g. hash, tenantId, companyId if applicable). Hash ~6 alphanumeric (exclude easily confused characters). |
| **Payment source value** | In transaction/payment table or payment-source table: new value **ONLINE_PATIENT_PAYMENT** (or equivalent) for how payment was taken. |
| **Payment provider reference ID** | Column (or equivalent) on transaction/payment to store provider’s reference ID for reconciliation. |
| **Payment application (allocation)** | Table or structure linking transaction/payment IDs to charge/procedure IDs if not already present. |

---

### 7. AWS (services and configuration)

| Create | Purpose |
|--------|---------|
| **New service / host for Payments API** | Run the Patient Payments API (e.g. ECS, Lambda + HTTP, or EC2). |
| **API Gateway** | Expose public endpoints for the UI; expose webhook endpoint that forwards to SQS (not directly to API). Rate limiting at Gateway. |
| **SQS queue** | Receive webhook payloads from Gateway; Payments API consumes and processes (PaymentCompleted). |
| **SQS DLQ** | Failed messages after retries; alert support for investigation. |
| **Security group (or equivalent)** | Restrict EMR API so only the Payments API (e.g. by internal IP) can call the specific EMR endpoints used for login, charges, POST transactions, receipt. |
| **Lambda** (optional) | If email receipt uses PDF attachment: build MIME + attach PDF, send via SES so API doesn’t block. |
| **CloudWatch and/or Datadog** | Logging, metrics (request timing, funnel, error tracking), and alerting. No PII/PCI in logs. |

---

### 8. Third-party / external

| Create | Purpose |
|--------|---------|
| **Payment provider account and integration** | Stripe, Square, or chosen provider; configure webhook URL (API Gateway), keys, and test/live mode. |
| **Email** | SES (with optional Lambda for attachments) or third-party mailer (e.g. Mailhog, Mailchimp); templates and sender configuration. |
