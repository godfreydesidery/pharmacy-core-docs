# Product Requirements Document: Pharmacy ERP

| | |
|---|---|
| **Product** | Pharmacy ERP: a pharmacy management system for multiple branches and multiple tenants |
| **Version** | 1.1 (Draft) |
| **Date** | 2 November 2026 |
| **Owner** | Godfrey Desidery |
| **Status** | For review |
| **UI reference** | `index.html` (clickable prototype of every module) |

---

## 1. Overview

Pharmacy ERP is a cloud-based system for running retail and wholesale pharmacies. It covers point of sale, inventory with batch and expiry control, purchasing, customers and insurance, accounting, and reporting. A single pharmacy runs it on one counter. A chain runs it across many branches. A SaaS operator hosts thousands of pharmacies on shared infrastructure.

The system is built to scale along three axes:

1. **Tenants.** Many independent pharmacy businesses share one platform, and each tenant's data is isolated.
2. **Branches.** One tenant can operate many branches and warehouses, with stock transfers and consolidated reporting.
3. **Load.** The system handles peak POS traffic, large catalogues and years of transaction history without slowing down.

### 1.1 Problem

Most small and mid-sized pharmacies still run on paper, spreadsheets or desktop software that serves a single shop. This causes:

- **Expired stock losses.** Batches are not tracked, so medicines expire on shelves unnoticed.
- **Stock-outs of fast movers.** There is no reorder logic and no visibility across branches.
- **Revenue leakage.** Sales go unrecorded, discounts are not controlled, and cash does not reconcile.
- **Slow insurance claims.** Claims are prepared by hand and often rejected.
- **Compliance gaps.** Controlled-substance registers, prescription records and fiscal receipts are kept poorly.
- **No business view.** Owners cannot see profit, margins or branch performance in real time.

### 1.2 Vision

> One reliable system that lets any pharmacy, from a single counter to a national chain, sell faster, never lose stock to expiry, and know its numbers every day.

---

## 2. Goals & Success Metrics

| Goal | Metric | Target (12 months after launch) |
|---|---|---|
| Faster checkout | Median POS transaction time | < 30 seconds |
| Less expiry waste | Value of expired stock written off | −50% vs. baseline |
| Fewer stock-outs | Stock-out rate on top-100 SKUs | < 2% |
| Accurate cash | Daily cash variance | < 0.5% of sales |
| Adoption | Active tenants | 500 |
| Adoption | Active branches | 1,500 |
| Reliability | Monthly uptime (core API) | ≥ 99.9% |
| Performance | POS API p95 latency | < 300 ms |
| Satisfaction | Customer NPS | ≥ 40 |

### 2.1 Non-goals (v1)

- A hospital pharmacy that integrates with ward or EMR workflows (v2 or later).
- A consumer e-commerce storefront or delivery app. v1 only exposes APIs for one.
- Full payroll and HR.
- Manufacturing or compounding batch production.

---

## 3. Users & Personas

| Persona | Description | Key needs |
|---|---|---|
| **Owner / Admin** | Owns one or more pharmacies | Profit visibility, branch comparison, user control, audit |
| **Pharmacist** | Licensed dispenser | Fast dispensing, prescription capture, drug info, controlled-drug register |
| **Cashier** | Front-counter sales | Quick search, barcode scanning, multiple payment methods, receipts |
| **Store Keeper** | Receives and manages stock | Goods receiving, batch/expiry entry, stock counts, transfers |
| **Accountant** | Manages finances | Expenses, payables/receivables, ledger, tax reports |
| **Branch Manager** | Runs one branch | Branch dashboard, approvals, shift reconciliation |
| **Platform Operator** | SaaS provider staff | Tenant onboarding, billing, support, monitoring |

---

## 4. Scope: Functional Requirements

Priority: **P0** is required for launch, **P1** should ship soon after launch, **P2** comes later.

### 4.1 Authentication, Tenancy & Access Control

| ID | Requirement | Priority |
|---|---|---|
| AUTH-1 | Sign in with email or username and password. Passwords are hashed with Argon2id or bcrypt. | P0 |
| AUTH-2 | Optional 2FA (TOTP). Can be made mandatory for Admin roles. | P1 |
| AUTH-3 | Role-based access control with permissions per module and action (view / create / edit / delete / approve). | P0 |
| AUTH-4 | Default roles are Admin, Pharmacist, Cashier, Store Keeper and Accountant. Tenants can also define custom roles. | P0 |
| AUTH-5 | Users can be limited to one or more branches. | P0 |
| AUTH-6 | Session timeout after inactivity (configurable, default 15 min). Forced logout and session revocation. | P0 |
| AUTH-7 | Every tenant's data is isolated. No query can read or write across tenants. | P0 |
| AUTH-8 | Single sign-on (OIDC/SAML) for enterprise tenants. | P2 |
| AUTH-9 | A quick PIN switches the user at a shared POS terminal. | P1 |

### 4.2 Dashboard

| ID | Requirement | Priority |
|---|---|---|
| DASH-1 | KPI cards: total sales, purchases, profit, customers and low-stock count, each with the change against the previous period. | P0 |
| DASH-2 | Sales trend chart for 7 days, 30 days and 12 months. | P0 |
| DASH-3 | Top-selling medicines, stock summary (in stock / low / out / expiring), expiry alerts, recent transactions and payment overview. | P0 |
| DASH-4 | Branch filter. Admins see figures for all branches combined. | P0 |
| DASH-5 | Dashboard figures are no more than 5 minutes old. Values come from pre-aggregated data, never from live scans of transactions. | P0 |

### 4.3 Sales & Point of Sale

| ID | Requirement | Priority |
|---|---|---|
| POS-1 | Search medicines by name, generic name, barcode or batch. Results in under 200 ms. | P0 |
| POS-2 | Barcode scanner input (keyboard wedge). | P0 |
| POS-3 | Cart with quantity changes, per-line and whole-invoice discounts (capped by role), and automatic tax calculation. | P0 |
| POS-4 | Batches are picked automatically by **FEFO** (first-expiry, first-out). Expired batches can never be sold. | P0 |
| POS-5 | Payment methods: Cash, Card, Mobile Money (e.g. M-Pesa, Tigo Pesa, Airtel Money), Insurance and Credit. Split payments allowed. | P0 |
| POS-6 | Cash tendered and change calculation. | P0 |
| POS-7 | Receipts print on 58/80 mm thermal printers and A4, and can be sent by SMS or email. | P0 |
| POS-8 | Fiscal receipts are issued through the country's tax authority device or API (e.g. TRA VFD/EFD in Tanzania), using a pluggable adapter per country. | P0 |
| POS-9 | Prescription-only (Rx) items require prescriber details and a prescription reference or image before the sale completes. | P0 |
| POS-10 | Hold and recall sales. | P1 |
| POS-11 | **Offline mode**: the POS keeps selling during an internet outage using a local cache and a queue, then syncs when the connection returns (see §6.4). | P0 |
| POS-12 | Shift open/close with cash count, X-reports and Z-reports, and variance recording. | P0 |
| POS-13 | Sales returns against the original invoice. Restocking depends on the reason. Refunds need approval above a threshold. | P0 |
| POS-14 | Customer loyalty points. | P2 |

### 4.4 Inventory

| ID | Requirement | Priority |
|---|---|---|
| INV-1 | Stock is held per **product → batch → location (branch/warehouse/shelf)**. | P0 |
| INV-2 | Each batch stores its expiry date, cost price, selling price and supplier. | P0 |
| INV-3 | Reorder level per product per branch. Low-stock alerts. | P0 |
| INV-4 | Expiry management in buckets (expired, ≤30, 31–90, >90 days), with alerts and a write-off workflow. | P0 |
| INV-5 | Stock adjustments (damage, theft/loss, count correction, expiry write-off) with a reason code and approval. | P0 |
| INV-6 | Stock counts (full and cycle counts), with a variance report before posting. | P1 |
| INV-7 | Stock transfers between branches: request → approve → dispatch → receive. Stock in transit is tracked. | P0 |
| INV-8 | Every stock movement is recorded in an append-only **stock ledger**. Current stock is calculated from it. | P0 |
| INV-9 | Unit conversions (e.g. box → strip → tablet). Selling in loose units is supported. | P1 |
| INV-10 | Suggested purchase quantities based on sales velocity and lead time. | P2 |
| INV-11 | Controlled-substance register with running balances and two-person verification. | P1 |

### 4.5 Purchasing

| ID | Requirement | Priority |
|---|---|---|
| PUR-1 | Purchase orders, with a workflow from Draft → Approved → Ordered → Partial → Received → Closed / Cancelled. | P0 |
| PUR-2 | Goods Received Note (GRN) against a PO. Batch, expiry and quantity are captured for each line. Partial receipts allowed. | P0 |
| PUR-3 | Supplier invoice matching (PO ↔ GRN ↔ invoice). | P1 |
| PUR-4 | Purchase returns to the supplier (expired, damaged, recalled), with a credit note. | P0 |
| PUR-5 | POs are emailed or exported as PDF to the supplier. | P1 |
| PUR-6 | Product recall handling: affected batches are flagged everywhere and blocked from sale. | P1 |

### 4.6 Customers & Insurance

| ID | Requirement | Priority |
|---|---|---|
| CUS-1 | Customer profiles with type (Retail, Wholesale, Insurance), contact details, credit limit and balance. | P0 |
| CUS-2 | Purchase history and dispensing history for each customer. | P0 |
| CUS-3 | Insurance schemes (e.g. NHIF, private insurers) with copay rules and the list of covered items. | P1 |
| CUS-4 | Insurance claim batches: generate, export or submit, then track status (submitted / approved / rejected / paid). | P1 |
| CUS-5 | Customer statements and reminders for overdue balances by SMS or email. | P1 |

### 4.7 Suppliers

| ID | Requirement | Priority |
|---|---|---|
| SUP-1 | Supplier profiles with contact details, TIN, payment terms and balance. | P0 |
| SUP-2 | Supplier statement, ageing of payables, and purchase history. | P0 |
| SUP-3 | Supplier performance: lead time, fill rate and return rate. | P2 |

### 4.8 Accounting

| ID | Requirement | Priority |
|---|---|---|
| ACC-1 | Expenses recorded by category, with attachments and approval. | P0 |
| ACC-2 | Payments received from customers and insurers, and payments made to suppliers, allocated against invoices. | P0 |
| ACC-3 | Double-entry general ledger. Sales, purchases, returns and payments post journal entries automatically. | P0 |
| ACC-4 | Chart of accounts (a default template that tenants can edit). | P0 |
| ACC-5 | Cash, bank and mobile-money accounts, with reconciliation. | P1 |
| ACC-6 | Period close. Posted entries cannot be changed; corrections are made by reversing entries. | P1 |
| ACC-7 | Export to external accounting tools (CSV; QuickBooks/Xero connectors later). | P2 |

### 4.9 Reports

| ID | Requirement | Priority |
|---|---|---|
| REP-1 | Sales report by day, product, category, cashier, payment method and branch. | P0 |
| REP-2 | Purchase report by supplier and product. | P0 |
| REP-3 | Stock valuation at cost and at retail, and stock movement report. | P0 |
| REP-4 | Profit & loss, and gross margin by product and category. | P0 |
| REP-5 | Expiry report and write-off report. | P0 |
| REP-6 | Tax / VAT report. | P0 |
| REP-7 | Every report can be exported to PDF and Excel/CSV. | P0 |
| REP-8 | Scheduled reports by email (daily summary at close). | P1 |
| REP-9 | Large reports are generated asynchronously and the user is notified when ready. They never block the UI. | P0 |

### 4.10 Master Data

| ID | Requirement | Priority |
|---|---|---|
| MD-1 | Medicines: brand name, generic name (WHO INN), **ATC code**, dosage form, strength, manufacturer, **GS1 GTIN** barcode and internal SKU, Rx/OTC/controlled class, regulator registration number (e.g. TMDA), storage conditions, tax rate and reorder level. | P0 |
| MD-1a | Scanning **GS1 DataMatrix** (2D) codes captures GTIN, batch/lot, expiry and serial number in one scan, both at goods receiving and at POS. | P1 |
| MD-2 | Categories, units and dosage forms, manufacturers. | P0 |
| MD-3 | Bulk import and export from CSV/Excel, with validation and an error report. | P0 |
| MD-4 | Optional shared **global drug catalogue** that tenants copy from, so they don't have to type products in. | P1 |
| MD-5 | Drug interaction and allergy warnings at dispensing, using a licensed drug database. | P2 |

### 4.11 Settings & Administration

| ID | Requirement | Priority |
|---|---|---|
| SET-1 | Pharmacy profile: name, logo, license number, TIN, address, currency, time zone and date format. | P0 |
| SET-2 | Branch management. | P0 |
| SET-3 | Invoice and PO numbering per branch, tax rates, receipt layout and footer. | P0 |
| SET-4 | Notification preferences (low stock, expiry, daily summary, approvals). | P1 |
| SET-5 | Data export of the tenant's full dataset. | P1 |
| SET-6 | Audit log: who did what and when, with before and after values for sensitive actions. | P0 |

### 4.12 Platform (SaaS operator)

| ID | Requirement | Priority |
|---|---|---|
| PLT-1 | Tenant onboarding wizard (business details, first branch, admin user, catalogue import). | P0 |
| PLT-2 | Subscription plans with limits (branches, users, SKUs) and feature flags. | P1 |
| PLT-3 | Billing integration (invoices, mobile-money/card payments, dunning). | P1 |
| PLT-4 | Operator console: tenant list, usage, health, and support impersonation that is consent-based and audited. | P1 |

---

## 5. Key User Flows

### 5.1 Counter sale (Cashier)
1. Scan a barcode or search for the item, which is added to the cart. The FEFO batch is picked automatically.
2. If the item is Rx, the system asks for prescription details.
3. Apply a discount within the role's limit. Tax is calculated.
4. Select the payment method or methods and enter the amount tendered. Change is shown.
5. **Complete sale.** Stock is decremented, a fiscal receipt is issued and printed, and journal entries are posted.

### 5.2 Receiving stock (Store Keeper)
1. Open the approved PO, then choose **Receive Goods**.
2. Enter the received quantity, batch number and expiry date for each line.
3. Record discrepancies (short, damaged, wrong batch).
4. Post the GRN. Stock increases, the payable is created and the PO status updates.

### 5.3 Expiry handling (Pharmacist / Manager)
1. A daily job flags batches entering the 90, 60 and 30-day windows, and notifications are sent.
2. The user decides for each batch: discount it, transfer it to a faster-moving branch, or return it to the supplier.
3. On expiry the batch is blocked from sale automatically, then written off with approval.

### 5.4 Shift close (Cashier / Manager)
1. The cashier counts cash by denomination.
2. The system compares the count with expected cash, mobile money and card totals.
3. The variance is recorded. The manager approves and the Z-report is generated.

---

## 6. Non-Functional Requirements

### 6.1 Scalability targets

| Dimension | v1 target | Design headroom |
|---|---|---|
| Tenants | 500 | 10,000 |
| Branches | 1,500 | 30,000 |
| Concurrent POS terminals | 3,000 | 50,000 |
| Sales transactions / day (platform) | 500,000 | 10,000,000 |
| SKUs per tenant | 20,000 | 200,000 |
| Data retention | 7 years (configurable to local law) | — |

### 6.2 Performance

- POS search: p95 < 200 ms.
- POS sale commit: p95 < 300 ms.
- Dashboard load: p95 < 1.5 s.
- Standard reports for up to 90 days: < 5 s. Longer ranges run asynchronously.
- The POS page loads in < 3 s on a 3G connection and on low-end Android tablets.

### 6.3 Availability & reliability

- Core API uptime ≥ 99.9% each month.
- RPO ≤ 5 minutes and RTO ≤ 1 hour.
- Point-in-time database recovery. Daily backups are kept for 35 days and tested restores run every quarter.
- Zero-downtime deploys (rolling or blue-green).

### 6.4 Offline-first POS

- The POS caches the catalogue, prices, current stock snapshot and customers locally (IndexedDB, or SQLite on desktop/mobile).
- Sales made offline are written to a local queue. Each sale carries a **client-generated idempotency key** (UUID), so the server never double-posts on retry.
- Offline invoice numbers come from a **per-terminal number block** reserved in advance, so they never collide.
- During sync, stock conflicts (e.g. overselling) are flagged for manager review, never silently dropped.
- Offline duration is capped (configurable, default 72 h). Fiscal receipts follow local rules for offline issuance.

### 6.5 Security & compliance

- TLS 1.2+ everywhere. Data is encrypted at rest (AES-256).
- Tenant isolation is enforced in the data layer with row-level security, not only in application code.
- Secrets are kept in a managed vault. API keys never reach the client.
- Follows OWASP ASVS Level 2. Penetration test before GA and once a year.
- Personal and health data handling follows applicable data-protection law (e.g. Tanzania PDPA 2022, Kenya DPA 2019, GDPR for EU tenants). Covers consent, the right to access and export, and retention.
- Audit trail for: price changes, discounts above threshold, voids, returns, stock adjustments, controlled-drug movements and permission changes.
- Prescription and controlled-substance records meet the local pharmacy regulator's requirements (e.g. TMDA / Pharmacy Council in Tanzania).

### 6.6 Usability & accessibility

- Responsive web app for desktop, tablet and phone. The POS can be fully operated by keyboard.
- Meets WCAG 2.1 AA.
- Localisation: English and Swahili at launch, with an i18n framework for more languages. Currency, number and date formats follow the tenant's settings.

### 6.7 Observability

- Structured logs with tenant ID, branch ID and request ID.
- Metrics and distributed tracing (OpenTelemetry).
- Alerts on error rate, latency SLOs, queue backlog and sync failures.

### 6.8 Scalability engineering

These are hard rules for engineering. They exist so the targets in §6.1 can be reached by adding capacity, not by rewriting the system.

| Area | Requirement |
|---|---|
| **Stateless services** | No session or user state in application memory, so any node can serve any request (12-Factor App). Sessions are opaque, revocable tokens whose hashes live in PostgreSQL (ADR 0008); what instances share (rate limits, job leases, the live stream, idempotency keys) lives in PostgreSQL too. |
| **Horizontal autoscaling** | API and worker pods scale on CPU, request rate and queue depth. The minimum is 2 replicas per service across 2 or more availability zones. |
| **Database scaling** | Writes go to the primary and reads go to replicas. Connection pooling (PgBouncer). Large tables (`stock_movement`, `sale_line`, `journal_line`, `audit_log`) are partitioned by month. Tenants can be sharded by `tenant_id` (§8.4). |
| **Caching** | Small read caches per instance, each with a short TTL (settings, a tenant's time zone, the edge's access checks), cleared by an event where the data changes. HTTP caching (ETag) for read-only lookups. A shared cache such as Redis is added only when measurements call for it (ADR 0008). |
| **Asynchronous work** | Anything slower than ~200 ms or calling a third party goes through the message broker: fiscal submission, SMS/email, report generation and claims. Retries use exponential backoff, with a dead-letter queue. |
| **Idempotency** | Every write endpoint accepts an `Idempotency-Key` header. Message consumers are idempotent, so delivering the same message twice is safe. |
| **Concurrency control** | Stock is reserved with optimistic locking (version column) or row-level locks scoped to one batch. A global lock is never taken. |
| **Pagination & limits** | Every list endpoint uses cursor pagination (max page size 100). Bulk endpoints are capped and run as asynchronous jobs. |
| **Rate limiting & fair use** | Limits per tenant and per API key at the gateway, so one tenant cannot degrade the others (noisy-neighbour protection). |
| **Backward-compatible change** | API versioning (`/v1`). Database migrations follow expand → migrate → contract, so deploys need no downtime. |
| **Data lifecycle** | Hot data (≤ 18 months) stays in OLTP. Older data is archived to cheaper storage but remains queryable for reports. Retention follows local law. |
| **Capacity validation** | Load tests (k6/Gatling) run in CI before each major release, at **2× the current peak**. Releases are blocked if the p95 SLOs in §6.2 regress. |
| **Resilience** | Timeouts, retries and circuit breakers on every external call. The POS degrades gracefully (offline mode) when a dependency is down. |

---

## 7. Standards & Compliance

The system must conform to the standards below. "Must" items are launch criteria. "Should" items are on the roadmap. Country-specific items are delivered through the adapter layer (§8.6).

### 7.1 Pharmacy practice & regulatory

| Standard / regulation | How it applies | Level |
|---|---|---|
| **WHO/FIP Good Pharmacy Practice (GPP)** | Dispensing workflow, prescription capture, patient counselling notes, pharmacist sign-off for Rx items | Must |
| **Good Distribution / Storage Practice (GDP/GSP)** | Batch traceability, FEFO, storage-condition fields, cold-chain flags, recall handling, quarantine of damaged or expired stock | Must |
| **National medicines regulator** (e.g. **TMDA**, Pharmacy Council of Tanzania) | Product registration numbers, controlled-substance register, inspection-ready records, adverse-event reporting export | Must |
| **WHO ATC/DDD classification** | Every product has an ATC code for consistent categories and analytics | Must |
| **WHO INN** | Generic names follow International Nonproprietary Names | Must |

### 7.2 Product identification & traceability

| Standard | How it applies | Level |
|---|---|---|
| **GS1 GTIN** (EAN-13 / GTIN-14) | Primary product identifier and barcode | Must |
| **GS1 DataMatrix & Application Identifiers** (01 GTIN, 10 batch, 17 expiry, 21 serial) | Scan once to capture full batch data at GRN and POS | Should |
| **GS1 EPCIS 2.0** | Export traceability events for national track-and-trace programmes | Should |

### 7.3 Health data interoperability

| Standard | How it applies | Level |
|---|---|---|
| **HL7 FHIR R4** | Public interoperability API exposes `Medication`, `MedicationRequest` (e-prescriptions in), `MedicationDispense` (dispensing out), `Patient`, `Coverage`, `Claim`, `ClaimResponse` | Should (Phase 3) |
| **ICD-10 / ICD-11** | Diagnosis codes on insurance claims where the payer requires them | Must (for claims) |
| **Payer formats** (e.g. NHIF claim schema) | Claim export and submission through the insurance adapter | Must (for claims) |

### 7.4 Security & privacy

| Standard | How it applies | Level |
|---|---|---|
| **ISO/IEC 27001** | Information Security Management System for the platform. Controls mapped to Annex A | Must (certification target: 12 months after GA) |
| **ISO/IEC 27701** | Privacy information management on top of 27001 | Should |
| **SOC 2 Type II** | For enterprise and international customers | Should |
| **OWASP ASVS L2** & **OWASP Top 10** | Secure development requirements. SAST, DAST and dependency scanning in CI | Must |
| **Session authentication** (ADR 0008) | Opaque bearer tokens, stored only as SHA-256 hashes, revocable at once (sign-out, forced logout, password change); idle sign-out; step-up re-authentication for sensitive changes; TOTP and passkeys | Must |
| **OAuth 2.1 / OpenID Connect** | Single sign-on, and tokens for third-party API clients, added as another way to create a session (ADR 0008) | Could (later) |
| **NIST SP 800-63B** | Password and MFA policy (length rather than complexity rules, breached-password check, TOTP) | Must |
| **PCI DSS v4.0** | Card data never touches our servers. Use tokenised or P2PE terminals and hosted payment pages to stay at SAQ-A / P2PE scope | Must |
| **Data-protection law** (Tanzania **PDPA 2022**, Kenya **DPA 2019**, **GDPR** where applicable) | Lawful basis, consent, data-subject rights (access, export, erasure where lawful), breach notification, data residency options | Must |
| **ALCOA+ data integrity** | Records are Attributable, Legible, Contemporaneous, Original and Accurate. Audit trail is immutable. No hard deletes of business records | Must |

### 7.5 Financial & fiscal

| Standard | How it applies | Level |
|---|---|---|
| **Double-entry bookkeeping, IFRS for SMEs–aligned** | Chart of accounts, accrual postings, period close, reversal-only corrections | Must |
| **Tax authority fiscalisation** (e.g. **TRA VFD/EFD**, KRA eTIMS) | Every sale produces a fiscal receipt through a country adapter | Must |
| **ISO 4217** | Currency codes. Amounts stored as integer minor units or fixed-point decimals | Must |

### 7.6 Engineering & data conventions

| Standard | How it applies | Level |
|---|---|---|
| **OpenAPI 3.1** | Every REST endpoint is documented. The spec is the contract, with contract tests in CI | Must |
| **RFC 9457 Problem Details** | Standard JSON error format for all APIs | Must |
| **ISO 8601 / RFC 3339** | All timestamps stored in UTC and sent with an offset. Shown in the tenant's time zone | Must |
| **ISO 3166 / ISO 639 / Unicode CLDR** | Country, language and locale formatting | Must |
| **UTF-8** | Encoding everywhere | Must |
| **UUIDv7** | Public identifiers. Sortable by time and safe to generate offline on the POS | Must |
| **Semantic Versioning 2.0** | API and client releases | Must |
| **CloudEvents 1.0** | Envelope for domain events and outbound webhooks | Should |
| **OpenTelemetry** | Logs, metrics and traces | Must |
| **WCAG 2.1 AA** | Accessibility of all user interfaces | Must |
| **ISO/IEC 25010** | Quality model used for acceptance criteria (performance efficiency, reliability, security, maintainability, usability) | Should |

### 7.7 Compliance verification

- A compliance matrix links each standard to its requirement IDs, test cases and evidence. It is kept in the repo and reviewed every release.
- Automated checks in CI: OpenAPI lint, accessibility tests (axe), SAST/DAST, dependency CVE scan, and tenant-isolation tests.
- An external penetration test and an access review every year. A data-protection impact assessment (DPIA) before launch in each new country.

---

## 8. System Architecture (Proposed)

### 8.1 Principles

- **Modular monolith first, split later.** Clear module boundaries (Sales, Inventory, Purchasing, Accounting, Identity, Reporting) inside one deployable unit. A module is extracted into its own service only when its scale or team ownership demands it.
- **Stateless API nodes** behind a load balancer, which scale horizontally.
- **Event-driven side effects.** A sale emits events such as `SaleCompleted` and `StockMoved`. Consumers update ledgers, aggregates, notifications and fiscal submission asynchronously, using the **transactional outbox** pattern.
- **Separate read models** for dashboards and reports (CQRS-lite), so heavy queries never slow the POS.

### 8.2 Components

```
 Clients: Web app (SPA/PWA) · POS (PWA, offline) · Mobile · Public API
                │
         API Gateway / LB  (auth, rate limiting, tenant resolution)
                │
   ┌────────────┴─────────────────────────────────────────┐
   │  Application (stateless, horizontally scaled)        │
   │  Identity · Catalogue · Sales/POS · Inventory ·      │
   │  Purchasing · Customers · Accounting · Settings      │
   └───────┬───────────────┬───────────────┬──────────────┘
           │               │               │
     PostgreSQL       Redis (cache,     Message broker
     (primary +       sessions, locks)  (Kafka / RabbitMQ)
      read replicas)                          │
           │                    ┌─────────────┼──────────────┐
           │               Workers:  Reporting · Notifications ·
           │               Fiscal adapter · Insurance claims · Sync
           │                          │
     Object storage            Analytics store
     (receipts, Rx images,     (columnar warehouse for
      exports)                  reports & dashboards)
```

This is the target at scale. As built (ADRs 0001, 0006, 0008): one modular monolith run as several instances behind a load balancer; sessions, rate limits, job leases, idempotency keys and the live stream's fan-out in PostgreSQL; events through the transactional outbox (Spring Modulith) rather than a broker. Redis and a broker come in only when measurements call for them, without changing the API.

### 8.3 Suggested stack (non-binding)

| Layer | Option |
|---|---|
| Frontend | React/Next.js or Angular, PWA with service worker, IndexedDB |
| Backend | Java (Spring Boot) or Node.js (NestJS), modular monolith |
| Database | PostgreSQL with row-level security (18 as built); read replicas |
| Cache | Per-instance read caches; Redis only if measurements call for it (ADR 0008) |
| Messaging | Kafka (high volume) or RabbitMQ (simpler ops) |
| Search | PostgreSQL trigram search to start; OpenSearch when the catalogue grows large |
| Analytics | ClickHouse or BigQuery for reporting read models |
| Storage | S3-compatible object storage |
| Infra | Containers on Kubernetes (or a managed container service), IaC with Terraform |
| CI/CD | Automated tests, migrations with Flyway/Liquibase, staged rollouts |

### 8.4 Multi-tenancy model

- **Default:** a shared database and shared schema. Every table has a `tenant_id` column, and PostgreSQL row-level security policies filter on it using the tenant ID set for each database session.
- **Enterprise tier:** a dedicated schema or database for each tenant, routed by a tenant directory. Large chains and tenants with data-residency requirements can be moved here.
- **Sharding path:** tenants can be spread across multiple database clusters by `tenant_id`, using the tenant directory for routing. No application changes are needed.

### 8.5 Core data model (simplified)

```
Tenant 1─* Branch 1─* Location
Tenant 1─* User *─* Role *─* Permission ; User *─* Branch
Product 1─* Batch 1─* StockLevel (per Location)
StockMovement (append-only: type, qty, batch, location, ref_doc)
Sale 1─* SaleLine (→ Batch) ; Sale 1─* Payment ; Sale 0─1 Prescription
PurchaseOrder 1─* POLine ; GRN 1─* GRNLine (→ Batch) ; SupplierReturn
Customer ; Supplier ; InsuranceScheme ; Claim 1─* ClaimLine
JournalEntry 1─* JournalLine (→ Account)
AuditLog ; Outbox
```

Key rules:
- Changes to `StockLevel` happen only through `StockMovement`, inside the same transaction.
- Money is stored as fixed-point decimals (or integer minor units). Floating point is never used.
- Business documents are never hard-deleted. They are voided or reversed.

### 8.6 Integrations

| Integration | Purpose | Priority |
|---|---|---|
| Fiscal device / tax authority API (e.g. TRA VFD) | Legal receipts | P0 |
| Mobile money (M-Pesa, Tigo Pesa, Airtel Money) | Payments at POS, subscription billing | P0 |
| SMS gateway | Receipts, reminders, OTP | P1 |
| Email (SMTP/transactional provider) | Reports, POs, statements | P0 |
| Insurance (NHIF and private insurers) | Eligibility check, claims | P1 |
| Card terminal | Card payments | P1 |
| Accounting export | QuickBooks / Xero | P2 |
| Public REST API + webhooks | Third-party apps, e-commerce | P1 |

All integrations sit behind adapter interfaces, so each country can supply its own adapters.

---

## 9. Release Plan

| Phase | Scope | Exit criteria |
|---|---|---|
| **MVP (Phase 1)** | Auth & RBAC, master data, POS (online), inventory with batches/FEFO/expiry, purchasing & GRN, customers, suppliers, basic expenses, core reports, single tenant with multiple branches | 5 pilot pharmacies trading daily for 4 weeks, no P0 bugs |
| **Phase 2** | Offline POS, fiscal integration, mobile money, shift management, stock transfers, double-entry ledger, audit log, multi-tenant SaaS onboarding | 50 tenants live; 99.9% uptime over 30 days |
| **Phase 3** | Insurance claims, controlled-drug register, stock counts, scheduled reports, public API, subscription billing | 500 tenants; claims rejection rate < 10% |
| **Phase 4** | Reorder suggestions, drug interaction checks, SSO, analytics warehouse, e-commerce APIs, more countries | — |

---

## 10. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Unreliable internet at branches | Lost sales | Offline-first POS with idempotent sync |
| Fiscal / regulatory rules differ by country | Blocks expansion | Country adapter layer; compliance review per market |
| Poor master data (duplicate or misspelled products) | Bad reports, wrong stock | Global catalogue, import validation, duplicate detection |
| Data leak between tenants | Severe trust and legal damage | Row-level security, automated isolation tests, pentests |
| Reporting load slows down POS | Slow checkout | Read replicas and a separate analytics store |
| Staff resist change | Low adoption | Simple UI, Swahili localisation, onboarding and training content |

---

## 11. Open Questions

1. Which countries are in scope for launch after Tanzania? This decides which fiscal and insurance adapters come first.
2. Pricing model: per branch, per terminal, or tiered by transaction volume?
3. Must we support hosting on the tenant's own premises for large chains, or only SaaS?
4. Which licensed drug database should provide interaction and allergy data, and at what cost?
5. Do wholesale tenants need price lists, credit terms and delivery notes in v1?

---

## 12. Appendix

### 12.1 Glossary

| Term | Meaning |
|---|---|
| **FEFO** | First-Expiry, First-Out: sell the batch that expires soonest first |
| **GRN** | Goods Received Note |
| **PO** | Purchase Order |
| **Rx** | Prescription-only medicine |
| **OTC** | Over-the-counter medicine |
| **VFD / EFD** | Virtual / Electronic Fiscal Device for tax-compliant receipts |
| **RLS** | Row-Level Security (database-enforced tenant isolation) |
| **RPO / RTO** | Recovery Point / Recovery Time Objective |

### 12.2 Module ↔ prototype mapping

| Module | Prototype route |
|---|---|
| Dashboard | `#/dashboard` |
| POS, Invoices, Sales Returns | `#/sales/pos`, `#/sales/invoices`, `#/sales/returns` |
| Purchase Orders, GRN, Purchase Returns | `#/purchase/orders`, `#/purchase/grn`, `#/purchase/returns` |
| Stock, Expiry, Adjustments | `#/inventory/stock`, `#/inventory/expiry`, `#/inventory/adjustments` |
| Customers, Suppliers | `#/customers`, `#/suppliers` |
| Expenses, Payments, Ledger | `#/accounting/expenses`, `#/accounting/payments`, `#/accounting/ledger` |
| Reports | `#/reports/sales` … `#/reports/expiry` |
| Master Data | `#/master/medicines`, `#/master/categories`, `#/master/units`, `#/master/manufacturers` |
| Users & Roles, Settings | `#/users`, `#/settings` |
