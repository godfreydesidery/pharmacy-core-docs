# Pharmacy Core: adversarial review, 23 Nov 2026

The whole system was reviewed: API, web app, database migrations, deployment and docs. Ten reviewers each took one view and tried to break the system from it:

- security: sign-in and sessions; isolation between pharmacies and branches; integrations and bulk data; the web app
- money and stock correctness
- a dishonest insider
- operations
- performance
- usability and accessibility
- conformance to standards

They worked by reading code. Nothing was changed, built against Railway, or sent to a remote service. Two small Node reproductions were run locally, and the web reviewers ran the repo's i18n check.

**How to read the findings**

- Every finding names the file and line where the problem is. `api/`, `web/` and `docs/` stand for `pharmacy-core-api`, `pharmacy-core-web` and `pharmacy-core-docs`.
- **Confirmed** means the failing path was traced in code. **Plausible** means one link (Railway behaviour, framework internals) could not be checked offline.
- Findings marked **✔** were checked again by hand before this report was written.
- Severity:
  - **Critical**: money or stock can be stolen today with one ordinary login.
  - **High**: a serious breach, loss or outage that is easy to cause.
  - **Medium**: real harm, but it needs more conditions.
  - **Low**: hardening or tidying.
- Where several reviewers found the same thing, it appears once, with the other IDs in brackets.

The reviewers' full working notes are in the session scratchpad. This file is the durable copy.

---

## 1. Summary

**What is strong.** Tenant isolation holds:

- Every tenant table has forced row-level security (RLS) that fails closed. The app role cannot bypass it.
- Tenant settings are transaction-local, and pooled connections never carry a tenant over.
- No reviewer found a way to read another pharmacy's rows through the API.
- Ledgers are append-only, enforced in the database. Postings are balanced and idempotent.
- Prices are always set by the server. Checkout is idempotent. FEFO allocation cannot oversell or sell expired stock.
- Webhooks are well guarded against SSRF.
- The web app has no XSS sinks for user data.

Most of what the chaos world tested really holds.

**What is weak.** Problems cluster in five places:

1. **Business controls inside a pharmacy.** Nothing needs a second person, an approval, a value limit or an alert. Default roles mix duties: the Pharmacist preset can sell, refund, adjust stock, edit products and set credit limits. Several paths let one login take cash or controlled drugs without a trace that anyone would notice (FRD-1 to FRD-8).
2. **Accounting edge cases.** These are multi-currency payments, refund methods, insurance re-adjudication, stock-takes and repacking. In each, the ledger and the sub-ledgers drift apart (MON-1 to MON-9).
3. **Account takeover paths.** Passkeys, phone and 2FA can be added with no re-authentication, and they survive a password reset. A passkey satisfies "Require two-factor". Root accounts have no 2FA by default (AUTH-1 to AUTH-3).
4. **Production hardening.** Defaults for dev keys, the log provider and Swagger only warn instead of refusing to start. The web server can be crashed by one URL. There are no backups or restore runbook, no job monitoring and no CI for the web app (OPS-1 to OPS-7).
5. **Scale.** One sale-number lock per branch is held through the whole checkout. Nothing limits total DB connections, and there are no DB timeouts. Static assets are never cached or compressed. Ledger reports scan the whole journal (PERF-1 to PERF-4).

**Counts.** About 170 distinct findings after merging duplicates. 2 are Critical, about 40 High, about 75 Medium and the rest Low.

### The 15 to fix first

| # | Finding | Why first |
|---|---|---|
| 1 | **FRD-1 ✔** A cashier can issue a voucher "paid by cash" for any amount, without taking money, then spend it | Direct theft, one login, invisible in the till |
| 2 | **MON-1 ✔** A voucher tendered in USD is reduced by the USD figure, so a TZS 50 voucher buys about TZS 132,000 | Direct theft, same day |
| 3 | **MON-2 ✔** The cashier can type any exchange rate at the till and on customer payments | Cash out of the drawer |
| 4 | **FRD-3 ✔** A MOBILE_MONEY or CARD payment sent without `channel` is auto-approved | Cashier keeps the cash; the drawer still balances |
| 5 | **OPS-3 / WEB-1 ✔** `GET /%E0%A4%A` kills the web server. After 5 restarts Railway stops restarting it | Anyone on the internet can take every till offline |
| 6 | **AUTH-3 ✔** Root has no 2FA unless a setting that is off by default is turned on. Root can also be reset by SMS | Root = every pharmacy |
| 7 | **AUTH-1 / AUTH-2 ✔** An unattended till lets anyone add a passkey, which survives every reset. A passkey also counts as 2FA | Permanent account takeover |
| 8 | **OPS-1 / OPS-2 ✔** Default dev keys and the `log` provider are accepted in production. Reset links and codes go to the logs | Takeover by anyone who can read logs or the DB |
| 9 | **ISO-1 / ISO-2 ✔** Any member can list, download and **delete** any attachment (prescription scans). A branch-limited cashier can read any sale by uid | Health data leak, destroyed evidence |
| 10 | **WEB-2 ✔** After an "uncertain" checkout, parking or reloading makes a new idempotency key, so the sale is recorded twice | Double charges and double stock deductions |
| 11 | **OPS-4** No backups, restore runbook or restore drill, although the PRD promises point-in-time recovery for 35 days | One mistake is unrecoverable |
| 12 | **OPS-6** `.railway/railway.ts` has no restart policy, and the Railway config cutover deadline is **1 Dec 2026** | The API stays down after a crash |
| 13 | **INT-1** A crafted 10 MB `.xlsx` import can expand in memory, and `ExitOnOutOfMemoryError` then kills the API for everyone | One staff member can take the API down |
| 14 | **INT-2 ✔** A pharmacy's SMTP host can be any internal address. "Send test" then works as a port scanner, and the saved password can be stolen | SSRF into Railway's private network |
| 15 | **PERF-3 ✔** Assets are never cached and nothing is compressed: 714 KB is downloaded on every load, instead of 210 KB once | Every mobile-data user, today |

---

## 2. Fix plan in batches

Each batch is about one PR per repo. Batches 1 and 2 are mostly small changes and should come before more pharmacies are signed.

### Batch 1: Stop the bleeding (small, urgent)

> **Status 23 Nov 2026: built and tested (branch `batch1-hardening` in api, web and docs; PRs open,
> not yet merged or deployed).**
>
> Two items changed from the plan:
> - **`log` providers:** they mask message bodies rather than stop the server starting, because
>   Railway has no SMS provider and would not start.
> - **Restart policy:** `railway/iac` has no option for it, so after the cutover it is set in the
>   Railway dashboard.
>
> Still open:
> - one-time upload URLs (part of INT-9)
> - removing the API's public domain
> - changing the live root email

| Fix | Findings |
|---|---|
| `serve-prod.mjs`: try/catch around decode (400); destroy upstream on client close; drop incoming `x-real-ip` / `x-forwarded-for` / `forwarded`; send CSP, `frame-ancestors 'none'`, HSTS, nosniff, Referrer-Policy; generic 502 text; proxy only `/api/` and `/actuator/health`; fix the hashed-asset regex to `-[A-Z0-9]{8,}\.(js\|css)$`; serve gzip/brotli | OPS-3, WEB-1, WEB-3, WEB-4, WEB-7, WEB-8, WEB-13, OPS-12, OPS-13, PERF-3 |
| Restart policy ALWAYS in both `railway.toml` **and** `.railway/railway.ts`; do the cutover before 1 Dec | OPS-6, OPS-3 |
| Refuse to start outside `local`/`test`/`sim` when any of these hold: encryption key or files signing secret is the default or shorter than 32 bytes; a `log` notification provider is active; `web-url` is localhost; `files.local.root` is relative; Swagger is enabled | OPS-1, OPS-2, AUTH-9, INT-4, ISO-12, STD-24 |
| Actuator: expose only `health` publicly; Swagger off by default | OPS-12, INT-19 |
| `require-2fa-for-root=true` by default; exclude root from phone reset | AUTH-3 |
| Vouchers: new permission `sales.voucher_issue` (not given to cashiers); voucher sales counted in shift expected cash; VOUCHER only in base currency | FRD-1, MON-1 |
| Exchange rate from the till ignored unless the user holds a permission, then limited to ±tolerance (reuse `Priced.checkPlausible`) | MON-2, FRD-13 |
| MOBILE_MONEY / CARD / BANK_TRANSFER need a channel once the pharmacy has registered methods | FRD-3 |
| `GET /sales/{uid}` and `paymentRequest(uid)` filtered by branch; files routes mapped to owner permissions + branch, delete needs a manage permission and is audited | ISO-2, ISO-1, INT-9 |
| Move `DEV_ACCOUNTS` out of the production bundle (`fileReplacements`); rotate any live account that used those passwords | WEB-6 |
| Root's live email: a real, unguessable address (not `root@pharmacy.local`) | OPS-19 |

### Batch 2: Accounts and sessions

> **Status 23 Nov 2026: built and tested (branch `batch2-accounts` in api, web and docs).**
>
> Done:
> - AUTH-1, AUTH-2, AUTH-4, AUTH-5, AUTH-6, AUTH-7, AUTH-8, AUTH-10, AUTH-11, AUTH-12, AUTH-15
> - ISO-3, ISO-4, ISO-7, INT-3, INT-20
> - the blocklist and Argon2 parts of STD-21
>
> Changed from the plan:
> - Every session is signed out when an email changes.
> - The minimum password length stays at 8, so a 15-character minimum for single-factor accounts is
>   left for the owner to decide.
>
> Moved to a later batch:
> - AUTH-13 (a revert link on email change)
> - AUTH-14 (lockout abuse: shared counters with the multi-instance work)
> - AUTH-16 (keyed hash for short codes)
> - AUTH-17 (idle limit taken from each request's pharmacy)

- **AUTH-1:** step-up check (password + TOTP, or a "recent sign-in" window of about 5 min) for passkey add/remove, phone verify, 2FA enable/disable and recovery codes.
- **AUTH-1:** delete passkeys on password reset, phone reset and recovery.
- **AUTH-1:** send notices for every security change.
- **AUTH-2:** record on the session which factors were proved, and enforce the policy on the session.
- **AUTH-10:** passkeys must require user verification (`UserVerificationRequirement.REQUIRED`).
- **AUTH-4:** SMS reset only for phones verified recently, plus an email confirmation or a hold.
- **AUTH-5:** count failed re-authentications. Rate-limit account routes per user.
- **AUTH-6:** check the old password or code before hashing the new one. Put a global semaphore around Argon2.
- **AUTH-7:** throttle on the matched handler, not on the raw URI prefix.
- **AUTH-8, ISO-3, INT-3:** never store raw tokens in `notifications.messages.body`. Render them at send time, or mask them. Mask in the data export too.
- **INT-20:** `access.*` templates cannot be overridden.
- **AUTH-11, ISO-8, WEB-14:** short single-use stream ticket instead of `?token=`. Close SSE streams on revoke.
- **ISO-4:** PLATFORM routes ignore `X-Tenant-Id`.
- **AUTH-12:** cancel open reset tokens on a new request, on password change and on email change. Cap requests per account.
- **AUTH-13:** revert link on email change.
- **AUTH-15, FRD-9:** grant only permissions you hold. Freeze the ADMIN role. Last-admin and owner guard on every membership change.
- **AUTH-16:** HMAC with a pepper for short codes.
- **STD-21:** NIST password rules (blocklist, 15 characters when there is no MFA, Argon2 m ≥ 19 MiB).

### Batch 3: Money and stock correctness

> **Status 23 Nov 2026: built and tested (branch `batch3` in api; `batch3-money` in web and
> docs).**
>
> Done:
> - MON-3 to MON-20, except the blind stock count (it moves to Batch 4 with the approvals).
> - MON-3 follows the owner's rule: a refund goes back the way the sale was paid, split by share.
>   A part-cash, part-account return is now proportional; before, it settled the open charge
>   first.
> - MON-12 adds `409 busy_try_again` on lock clashes.
> - MON-10 adds the generic `Idempotency-Key` header on 16 money routes.
> - MON-13 adds automatic resubmission of failed postings, `GET /finance/reconciliation` and a
>   ledger health check (kept out of readiness).
> - Batch cost stays at 2 decimal places. Rounding is posted, so stock at cost equals ledger
>   1200; a test asserts this after every step.
>
> Still open:
> - Supplier returns do not yet reduce a receipt's open amount.
> - Purchasing still rounds a foreign receipt's total to the cent once.
> - The existing scheduled jobs still run on every instance; `SingleRunner` is ready for them in
>   Batch 6.

- **MON-3:** refund method limited by how the sale was paid.
- **MON-4, MON-5:** insurance re-adjudication posts only the change. Returns on rejected claims go to the patient or reverse the write-off.
- **MON-6, FRD-12:** stock-take difference worked out against movements since the count started. Blind counts.
- **MON-7:** an insured sale's member must belong to the customer.
- **MON-8, MON-9:** repack and break-bulk average into existing batches and post differences. Units in transit included in averaging. 4-decimal batch cost.
- **MON-10, STD-11:** a generic idempotency filter (key + request hash, 422 when the body differs) on every money-moving POST.
- **MON-11:** lock the credit account during the credit check.
- **MON-12, PERF-1:** lock products in sorted order. Retry deadlocks (40P01) once. Map lock failures to 409/503.
- **MON-13:** nightly reconciliation of source documents against journal entries. Alert on incomplete event publications.
- **MON-14:** payment methods on expenses and supplier payments become an enum.
- **MON-15 to MON-19:** quote pricing, `excessToAccount` with push payments, cents on partial returns, supplier overpayment, negative remittance amounts.
- **MON-20, STD-12:** the tenant's time zone for every day boundary. Refuse future dates.

### Batch 4: Approvals and segregation of duties (needs owner decisions, see §4)

- One approval engine: request, then approval by a different user with a PIN, with thresholds per pharmacy. Used for refunds, adjustments and destructions, write-offs, transfers, period unlock, schedule changes, voucher issue and large discounts or promotions (FRD-2, 4, 5, 6, 8, 10, 15, 16, 17).
- New default role presets that separate selling from refunding, stock adjustment and catalogue editing.
- **FRD-2:** recipes enforced. No controlled input into a non-controlled output.
- **FRD-6:** a schedule change is a privileged, alerted action. The schedule at the time of sale is kept on the line.
- **FRD-11:** opening stock only during setup.
- **FRD-7:** open shift required (see §4: till shifts are currently hidden).
- **FRD-19:** owner alert digest for sensitive events.

### Batch 5: Integrations and bulk data

- **INT-1:** zip-bomb and cell-size limits for imports.
- **INT-2:** SMTP host public-address rule, allowed ports only, secrets dropped when the host changes, STARTTLS required.
- **INT-5:** enforce the SMS plan cap.
- **INT-6:** users cap counts invitations and reactivations.
- **INT-7, PERF-12:** asynchronous export to storage plus a signed link, one export at a time.
- **INT-8:** retention skips pharmacies under legal hold.
- **INT-10, ISO-5:** definer functions clamp dates to `now()` and enforce a minimum audit retention.
- **INT-11, WEB-11:** CSV formula neutralising.
- **INT-13:** seal the webhook secret; Standard Webhooks signing with a timestamp; rotation; cap of about 10 webhooks.
- **INT-15:** SIMULATOR only in sim/local.
- **INT-16:** erasure also removes `event_publication` rows and retries file deletes.
- **INT-17:** the export excludes root's notes.
- **ISO-6:** a former employer sees only the `former_*` snapshot.
- **ISO-7:** roles queries filtered by tenant.
- **ISO-9:** body-supplied branch checked.
- **ISO-10:** stronger permissions on exports and account reads.
- **ISO-11:** RLS on `access.session_pharmacies`.

### Batch 6: Operations

- **OPS-4:** backups off Railway, a restore runbook and a restore drill; files moved to S3/R2.
- **OPS-5:** key ids and a keyring so keys can be rotated; separate keys for TOTP and provider secrets.
- **OPS-7:** job last-run/outcome table, metrics (Prometheus/OTel), an error tracker and alerts.
- **OPS-8, INT-12:** advisory locks per job, `SKIP LOCKED` claims, and a "single instance" note in deployment.md until both are done.
- **OPS-9:** MigrationSafetyTest reads forced RLS from the real database and scans DO blocks.
- **OPS-10:** `lock_timeout` for migrations.
- **OPS-11:** migrate-only mode, so the API no longer holds the owner password.
- **OPS-14, STD-24:** web CI; deploys only from CI on `main`; image tagged with the git SHA.
- **OPS-15, STD-5, STD-6:** PDPA groundwork (see §4 on hosting).
- **OPS-16, OPS-17:** `npm ci`, a lock file generated on Linux, pinned images, Dependabot, CVE scan and an SBOM.

### Batch 7: Performance

- **PERF-1:** payment and fiscal calls outside the stock/number transaction; the sale number taken last.
- **PERF-2:** bounded async executor, global in-flight cap, role-level `statement_timeout` / `lock_timeout` / `idle_in_transaction_session_timeout`, scheduled resubmission of incomplete events.
- **PERF-4:** daily account-balance read model.
- **PERF-5:** cursor-based exports; capped or optional totals.
- **PERF-6:** `(tenant_uid[, branch_uid], uid)` indexes, checked with EXPLAIN on the QA copy.
- **PERF-7:** stop auditing child rows of append-only ledgers.
- **PERF-8:** product stock read model.
- **PERF-9:** cache membership, tenant and branch at the edge for 30 s.
- **PERF-10:** settings read once per checkout; JDBC batching.
- **PERF-11:** price-list item lookup by product.
- **PERF-13:** purge `reporting.processed_events`.
- **PERF-15:** `sale_lines(tenant_uid, batch_uid)` index.
- **PERF-16 to PERF-19:** N+1 queries, statements, dictionaries and fonts, report limits.

### Batch 8: Till robustness, usability and accessibility

- **WEB-2:** lock the cart after an uncertain checkout; keep the key with a parked sale.
- **UX-1:** block unpriced lines before Complete sale.
- **UX-2:** checkout problems shown in the Payment card, not toasts.
- **WEB-9:** push payment bound to its sale.
- **WEB-10:** "Exact" rounding.
- **WEB-5:** parked sales expire and hold no personal data.
- **WEB-12:** returnUrl only for the same user.
- **WEB-15:** dialog queue.
- One accessible Dialog component (A11Y-3) and one Field component (A11Y-1).
- Keyboard-operable rows (A11Y-2) and contrast tokens (A11Y-4).
- Error keys for the 87 untranslated reasons, with parameters (UX-3).
- Unsaved-form guard (UX-4), the suspended-pharmacy page (UX-8), staff-login forgot-password (UX-9), list error with retry (UX-10).
- The remaining browser `confirm()` calls (UX-7).

### Batch 9: Regulation and standards

- **STD-1, STD-2:** fiscal-ready design (VFD/EFD port, TIN/VRN, tax categories, input VAT).
- **STD-5:** PDPA per-person rights and consent.
- **STD-7, STD-8:** dispensing records (pharmacist sign-off, TMDA number, adverse-drug-reaction reporting).
- **STD-9:** OpenAPI generated from `@Access` and diffed in CI.
- **STD-10:** ETag/If-Match.
- **STD-13:** webhook signing.
- **STD-17:** money scale per currency.
- **STD-18:** GTIN check digit; batch numbers case-insensitive.

---

## 3. Findings by area

### 3.1 Insider fraud and missing controls (FRD)

The database protections are strong. The gap is in business controls. The PRD asks for approvals in POS-13, INV-5, INV-7, PUR-1 and ACC-1, and none are built.

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| FRD-1 ✔ | Critical | `POST /vouchers` needs only `sales.sell`. `paidBy` defaults to CASH, no money is taken, there is no value limit, and shift expected cash ignores vouchers | api/delivery/.../sales/PaymentsHandler.java:112-114; api/modules/sales/.../VouchersImpl.java:47-77; ShiftsImpl.java:129-151 | Own permission, counted in the shift, value cap plus approval, owner alert |
| FRD-2 | High | Repack and break bulk accept any inputs and outputs: 100 morphine in, 1 OTC unit out. No schedule check | api/modules/inventory/.../TransformationsImpl.java:52-200 | Enforce the recipe; no controlled input into a lower schedule; witness PIN |
| FRD-3 ✔ | High | A non-cash payment without `channel` is approved with any reference | api/application/pharmacy-core/.../ManualPaymentAuthorizations.java:39-48 | Require a channel; unique references; a reconciliation report |
| FRD-4 | High | Returns: refund method free, no time window, no approval, stock always goes back to sellable (also MON-3) | api/modules/sales/.../SaleServiceImpl.java:373-394; StockServiceImpl.java:153-186 | Refund by the original tender; return window; approval above a threshold; restock reason |
| FRD-5 | High | Adjustments marked DAMAGE or EXPIRED_WRITE_OFF bypass the destruction flow. The note is optional and the witness is free text | api/delivery/.../inventory/StockHandler.java:166-180; DestructionsImpl.java:65-66 | All write-offs through destruction; second-user PIN for controlled items and above a value |
| FRD-6 | High | `catalogue.manage` (held by the Pharmacist preset) can switch CONTROLLED to OTC, sell without a prescription, then switch back. The register refuses to open while the product is OTC. Prescription references can be reused | api/delivery/.../catalogue/CatalogueHandler.java:186-187; ControlledDrugsHandler.java:147-150 | Schedule change is privileged and alerted; schedule kept on the sale line; register built from movements |
| FRD-7 | High | Sales do not need an open shift. Refunds made outside a shift count against nobody | ShiftsImpl.java:72-86,156-176 | Setting to require an open shift (see §4) |
| FRD-8 | Med-High | Promotions of up to 100% can be created by `catalogue.manage`, deleted afterwards, and are not recorded on the sale line, so the discount limit is bypassed | api/modules/catalogue/.../PricingServiceImpl.java:242-246,287-291 | Cap at the maximum discount; never delete; store the promotion on the line |
| FRD-10 | Med | The period lock can be moved back by `finance.manage`, which the BRANCH_MANAGER preset holds | PeriodLockHandler.java:30-35; Permissions.java:113-114 | Own permission; forward-only without approval; owner alert |
| FRD-11 | Med | Opening stock can be added to a live branch at any time, with no ledger entry, and it re-costs the batch | StockHandler.java:148-164; StockServiceImpl.java:535,553 | Only during setup; no averaging; post to the ledger |
| FRD-12 | Med | Stock-takes are not blind and use a stale snapshot (see MON-6) | StockCountsImpl.java:131-145 | Blind count; approval |
| FRD-14 | Med | A customer's price plan or insurer applies with `sales.sell` only. Picking a wholesale or staff customer gives that price | PricingServiceImpl.java:429-446 | Permission when the plan is not Normal; member verification |
| FRD-15 | Med | Transfers: the same person can send and receive; no partial receipt; stock can stay "in transit" for ever | StockServiceImpl.java:362-470 | Line receipt with shortfall loss; a different receiver; ageing alert |
| FRD-16 | Med | Supplier returns are not tied to a goods receipt and the supplier never confirms them | SupplierReturnsImpl.java:45-60 | Pending until the credit note; link a receipt or get approval |
| FRD-17 | Med | The Pharmacist preset both sets credit limits and sells on credit. Write-offs have no threshold | Permissions.java:115-131; CustomerCreditImpl.java:97-123,270-291 | Move limits to finance; approval for write-offs |
| FRD-18 | Low-Med | One root admin alone can lift a legal hold, close a pharmacy and erase it, together with its audit trail | PlatformHandler.java:194-224 | Two root admins for hold lift and erase |
| FRD-19 | Low | Audit entries lack IP and session. Root reads are logged once an hour. No alerts on returns, adjustments or role changes | AuditWriter.java:47-62; RootActivity.java:23-44 | Add IP and session; log every root request; owner digest |
| FRD-20 | Low | Abandoned and parked carts are never seen by the server | web/src/app/features/pos/parked.ts | Report abandoned-cart and no-sale events |

### 3.2 Money and stock correctness (MON)

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| MON-1 ✔ | Critical | A foreign-currency VOUCHER tender spends the foreign figure. `onAccountMethod` excludes VOUCHER | api/modules/sales/.../SaleServiceImpl.java:268-270; api/application/checkout/.../CheckoutService.java:490-494 | VOUCHER in base currency only, or spend `baseAmount()` |
| MON-2 ✔ | High | The client's `exchangeRate` is accepted with no limit at the till and on receivables payments. USD 100 at 10,000 gives TZS 735,000 in change | SaleServiceImpl.java:245-250, 674-679; CustomerCreditImpl.java:453 | Recorded rate only, or permission plus ±25% |
| MON-3 | High | A return with refund method CREDIT, ACCOUNT_CREDIT, VOUCHER or INSURANCE posts to accounts 1100/2300/1110 with no sub-ledger entry | SaleServiceImpl.java:384; finance/.../Postings.java:99-100,221-233 | Allowed methods depend on the sale |
| MON-4 | High | Adjudicating a claim again posts the full rejected amount again, so 1110 drifts negative | api/modules/insurance/.../InsuranceImpl.java:532; CreditReactions.java:122-135 | Adjudicate once, or post only the change |
| MON-5 | Med-High | A return on a rejected and written-off claim credits 1110 again | InsuranceImpl.java:382-412 | Route to the patient or reverse the write-off |
| MON-6 | High | Stock-take applies counted − expected to stock that kept moving. Sales during the count become phantom losses, and some counts cannot be posted at all | api/modules/inventory/.../StockCountsImpl.java:135-145 | Net out movements since the start, or freeze |
| MON-7 | Med | An insured sale accepts any member, not the customer's own | CheckoutService.java:125-132, 372-380 | `member_not_of_customer` |
| MON-8 | Med | Break bulk and repack round cost to 2 dp, and a reused output batch is not averaged. Nothing reaches the ledger | TransformationsImpl.java:171-189; CheckoutService.java:292-314 | `batch.receive` averaging plus a posted difference; 4-dp cost |
| MON-9 | Med | Units in transit are excluded from averaging and valuation. Returns come back at the sale-time cost. Foreign cost is rounded to 2 dp | Batch.java:50-65; StockServiceImpl.java:532-536 | Include transit; post differences |
| MON-10 | Med | Only checkout is idempotent. Payments, returns, transfers, expenses, vouchers and write-offs can double on a retry (also STD-11) | api/delivery/.../sales/SalesHandler.java | Generic idempotency filter |
| MON-11 | Med | Two credit sales at once can exceed the credit limit (unlocked SUM) | CustomerCreditImpl.java:97-121 | Lock the account row |
| MON-12 | Med | Checkout locks products in the client's line order, so two tills can deadlock and the cashier sees a 500. The product lock covers every branch of the pharmacy (also PERF-1) | CheckoutService.java:174-203; StockServiceImpl.java:612,803-804 | Sorted lock order; key includes the branch; retry |
| MON-13 | Med | A failed after-commit ledger posting is invisible and retried only at restart. A late posting is moved past the period lock silently | LedgerReactions.java:198-205 | Reconciliation job; health indicator; backoff retry |
| MON-14 | Low-Med | Free-text payment methods on expenses: "CREDIT" credits customer receivables, and a typo posts to cash | ExpenseServiceImpl.java:61 | Enum of money accounts |
| MON-15 | Low | The insurance coverage quote ignores the plan, branch and promotion prices | CheckoutService.java:630-644 | Price the quote the same way as checkout |
| MON-16 | Low | `excessToAccount` with a confirmed push payment is always declined | CheckoutService.java:422-433 | Verify before trimming the tender |
| MON-17 | Low | Partial returns leave a cent over or under on the line | SaleServiceImpl.java:432-434 | The last return takes the remainder |
| MON-18 | Low | A supplier payment can settle the same goods receipt again and again | SupplierPaymentsImpl.java:153-176 | Track the open amount |
| MON-19 | Low | Remittance allocations accept negative amounts | InsuranceImpl.java:617 | Each amount must be > 0 |
| MON-20 | Low | Sales lists and goods-receipt filters cut days at UTC; the ledger uses the tenant's day. Future dates are accepted (also STD-12) | SaleServiceImpl.java:531-533 | TenantCalendar zone; refuse the future |

### 3.3 Sign-in, sessions and accounts (AUTH)

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| AUTH-1 | High | Passkey, phone and 2FA can be added or removed with the session alone. Passkeys survive password reset, phone reset and recovery. No notices | api/modules/access/.../PasskeyServiceImpl.java:96,275; AccountSecurityImpl.java:282; TwoFactorServiceImpl.java:102,116 | Step-up; wipe passkeys on reset; notices |
| AUTH-2 ✔ | High | "Require two-factor" is met by a password-only sign-in when the user has a passkey but no TOTP | AuthenticationServiceImpl.java:124-127; SessionIssuer.java:178 | Factors proved on the session |
| AUTH-3 ✔ | High | `require-2fa-for-root` defaults to false and is missing from the config and the runbook. Root is eligible for SMS reset. A root can recover another root with no step-up | SessionIssuer.java:61; AccountSecurityImpl.java:349-351 | Default true; no SMS reset for root; step-up |
| AUTH-4 | Med | SMS alone resets a password. Recycled numbers and SIM swaps are common | AccountSecurityImpl.java:340-423 | Recency check, email confirm or hold |
| AUTH-5 | Med | Change-password, email-change, 2FA disable and phone routes are not throttled and failures are not counted. With a stolen session, TOTP can be brute-forced in about 30 min | api/delivery/.../edge/EdgeFilter.java:254-260 | Count failures; per-user limit |
| AUTH-6 | Med | The new password is hashed before the old one is checked. Account routes have no concurrency cap, so Argon2 can be used for CPU and memory denial of service | AuthenticationServiceImpl.java:323; AccountSecurityImpl.java:387 | Check first; Argon2 semaphore |
| AUTH-7 | Med (Plausible) | The throttle matches prefixes of the raw URI, so `/api/v1/auth;x/…` or `%61uth` skips it | EdgeFilter.java:97,250 | Match the resolved handler |
| AUTH-8 | Med | Reset, invitation and recovery links and SMS codes are stored in plain text in `notifications.messages` and written to logs (also OPS-2) | notifications/.../NotifierImpl.java:121,136-139; LogProviders.java | Don't store tokens; mask logs |
| AUTH-10 ✔ | Med | Passkey user verification is PREFERRED and never checked, yet a passkey counts as both factors | PasskeyServiceImpl.java:115,194 | REQUIRED + check `isUserVerified` |
| AUTH-11 | Low-Med | The session token travels in the SSE URL (`?token=`). An SSE stream keeps running 30 min after revocation. The invitation token sits in a GET path | EdgeFilter.java:226-230; NotificationStreams.java:27 | Stream ticket; close on revoke |
| AUTH-12 | Low | Reset tokens are not cancelled by a new request, a password change or an email change. No cap per account | PasswordResetServiceImpl.java:94,137 | Cancel; cap 3 per hour |
| AUTH-13 | Low | Email change gives the old address no revert link and leaves other sessions open | AccountSecurityImpl.java:215-226 | Revert link; revoke sessions |
| AUTH-14 | Low | An anonymous attacker can keep any account locked (5 failures in 15 min; staff logins are guessable; counters are in memory) | SignInGuard.java | Progressive delay; lock per account+IP; root can unlock |
| AUTH-15 | Low | `users.manage` can grant any permission and edit the ADMIN role. The last-admin check exists only on "leave" (also FRD-9) | UserAdministrationImpl.java:168-205,245-262 | Grant only what you hold; freeze ADMIN; owner guard |
| AUTH-16 | Low | Recovery and SMS codes are hashed with unsalted SHA-256 | RecoveryCodes.java:39 | HMAC with a pepper |
| AUTH-17 | Low | `/auth/activity` is self-reported, so a script can keep a session alive. The idle limit follows the pharmacy the client names | AuthHandler.java:119 | Limit from each tenant request |

### 3.4 Isolation between pharmacies and branches (ISO)

No cross-pharmacy read path was found. The gaps are inside a pharmacy, or in defence in depth.

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| ISO-1 ✔ | High | Files: list, url, download and delete need only TENANT scope. A branch cashier can fetch prescription scans and **delete** invoices. Upload URLs can overwrite a finished file (also INT-9) | api/delivery/.../files/FilesHandler.java:100-129; LocalObjectStorage.java:64-79 | Owner-type permissions + branch; soft delete + audit; refuse PUT once COMPLETE |
| ISO-2 ✔ | Med | `GET /sales/{uid}` has no branch filter, so a branch-limited cashier reads any sale (customer, prescription, payments). The same holds for `paymentRequest(uid)` | SaleServiceImpl.java:342-343; PaymentsImpl.java:182 | Filter by `requireBranch()`; route-wide test |
| ISO-3 | Med | Live invitation links in the outbox body are leaked by the data export. `settings.manage` can then join as Admin. Templates can be overridden (also INT-3) | TenantDataService.java:54,166; InvitationServiceImpl.java:138-141 | Don't store tokens; mask the export; bind join to the invited email |
| ISO-4 | Med (Plausible) | PLATFORM routes accept `X-Tenant-Id`, so a root password-reset email (raw token) is stored in that pharmacy's scope, and the pharmacy can export it | AccessInterceptor.java:87; PasswordResetServiceImpl.java:88-103 | Refuse or clear the tenant on PLATFORM routes |
| ISO-5 | Med | SECURITY DEFINER functions trust dates from the caller: `drop_partitions_before('2100-01-01')` wipes all audit; `erase_tenant` with a future `p_now` skips the cooling-off (also INT-10, FRD-18) | platform/V3__tenant_erasure.sql:27,53; audit/V5__retention.sql:3,16 | Clamp to `now()`; retention floor |
| ISO-6 | Low-Med | A former employer sees the person's *current* name and email (export, cashier report) and their lock state | TenantDataService.java:189-191; StatementsHandler.java:119-121; AccountLocksImpl.java:42-43 | `coalesce(former_*, …)`; active members only |
| ISO-7 | Low | `GET /roles` lists the caller's roles from other pharmacies. The child-table policies are loose | UserAdministrationImpl.java:212; access/V3__security.sql:149-158 | Tenant filter; tighter WITH CHECK |
| ISO-9 | Low | A branch price set through the body ignores `allowedBranches`. Recall quarantine acts on every branch | PricingHandler.java:82-90; RecallsImpl.java:46 | `requireAllowedBranch(uid)` helper |
| ISO-10 | Low | Bulk customer export (phones, insurance numbers) needs only `partners.view`, which cashiers have. Some reads have no permission at all | ExportHandler.java:232-233; ReceivablesHandler.java:52-53 | `*.export` permissions |
| ISO-11 | Low | `access.session_pharmacies` has no RLS. `event_publication` holds every pharmacy's payloads. Platform message rows with links are visible with no tenant | access/V9__pharmacies_apart.sql:6,12 | RLS / definer access; no tokens in bodies |
| ISO-12 | Low | The privileged-role start-up check compares only the role name, and `allow-privileged-role` turns it off | PersistenceConfiguration.java:34,50 | `pg_has_role` + ownership checks |

### 3.5 Integrations, files, imports and exports, retention (INT)

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| INT-1 | High (Plausible) | `.xlsx` decompression bomb (fastexcel, 10 MB compressed) leads to OOM, and `ExitOnOutOfMemoryError` stops the API for all pharmacies. `/import/check` works too | api/delivery/.../imports/SheetFile.java:73-91; Dockerfile:30 | Limits on entry size and ratio; cell length cap |
| INT-2 ✔ | High | SMTP host is unrestricted: internal SSRF, a port scanner through "Send test", the saved password sent to a new host, STARTTLS not required | notifications/.../ProviderSettingsImpl.java:299-313; ProviderRouter.java:351 | WebhookTargets rules; allowed ports; drop secrets on change; require TLS |
| INT-5 ✔ | Med | The plan's `sms_per_month` is never enforced, so a Starter pharmacy can send unlimited SMS on the platform's account | Plan.java:92; TenantUsageAdapter.java:44 | Check the cap in `NotifierImpl` |
| INT-6 | Med | The users cap can be passed with many invitations or with reactivations (a missing `active` means true). Caps race on parallel creates | UserHandler.java:149-163; InvitationHandler.java:49 | Count invitations; check on admit and reactivate; advisory lock |
| INT-7 | Med | The full export holds one REPEATABLE READ transaction and a connection for the whole download, with no rate limit (also PERF-12) | TenantDataService.java:80-129 | Asynchronous job + signed link; one at a time |
| INT-8 ✔ | Med | A legal hold does not stop the nightly purge or the audit partition drop | platform/V4__retention.sql:73; audit/V5__retention.sql | Skip held pharmacies |
| INT-11 | Low | CSV formula injection in API exports (also WEB-11 in client exports) | api/delivery/.../api/Spreadsheet.java:178-180; web/src/app/core/format/format.ts:178-190 | Prefix `= + - @` with `'` |
| INT-13 | Low | Webhook secret stored in plain text; HMAC over the body only (replayable); no rotation; unlimited webhooks; sequential dispatch | integrations/domain/Webhook.java:27; WebhookSender.java | Seal; Standard Webhooks; cap; parallel |
| INT-14 | Low (Plausible) | DNS-rebinding race; NAT64 and 6to4 addresses not refused | WebhookTargets.java | Connect to the checked IP |
| INT-15 | Low | SIMULATOR payment adapter exists in production (registry flag only). `verify()` does not compare method or branch | PaymentAdapters.java:75-106; PaymentsImpl.java:204-220 | sim/local only; compare method and branch |
| INT-16 | Low | Erasure leaves the pharmacy's incomplete `event_publication` rows, which replay after a restart. Failed file deletes are only logged. Tax-record retention (about 5 years) may conflict | platform/V3__tenant_erasure.sql | Delete them; retry queue; confirm an export was taken |
| INT-17 | Low | The export includes root's internal notes (`status_reason`, legal-hold reason) | TenantDataService.java:121-126 | Explicit columns |
| INT-20 | Low | A pharmacy can rewrite `access.*` templates sent under the platform's sender, which allows phishing | TemplatesImpl.java:64-83 | Platform-only templates |

### 3.6 Web app (WEB)

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| WEB-1 ✔ | High | Crash on a malformed URL (= OPS-3, reproduced in Node) | web/scripts/serve-prod.mjs:55 | Batch 1 |
| WEB-2 ✔ | High | After an uncertain checkout, parking, reload, idle sign-out or a branch switch gives a new idempotency key, so the sale is created twice. Editing the cart and retrying returns the original sale, and the added item leaves unrecorded. Every 409 counts as "uncertain" | web/src/app/features/pos/pos.page.ts:257,846-862,1218,1332-1363; parked.ts:9-17 | Lock the cart; keep the key in ParkedSale; the API compares a body hash |
| WEB-3 | Med | The proxy forwards the client's `X-Real-IP`, and the API trusts it from any private peer, so IP throttles can be spoofed unless the edge overwrites the header | serve-prod.mjs:34-36; api/.../ClientAddress.java:34-42 | Strip in the proxy; trust a configured CIDR only |
| WEB-4 | Med | No security headers on the web origin, and the bearer token is in localStorage (= OPS-13) | serve-prod.mjs:68-71; session.store.ts:46-50 | Batch 1; later an HttpOnly cookie |
| WEB-5 | Med | Parked sales keep the customer's name, phone, insurance number and medicines in localStorage for ever, even after sign-out. The token survives closing the browser | parked.ts:9-54; sign-out.ts:357-385 | Expiry; store only uids; "shared PC" mode |
| WEB-6 ✔ | Med | Root, admin and cashier dev passwords are in the production JS (`Root#2026` found in a dist chunk) | web/src/app/features/auth/sign-in.page.ts:20-24 | `fileReplacements`; rotate |
| WEB-7 | Med | The proxy never closes the upstream connection when the client leaves, so orphan SSE connections build up (reproduced) | serve-prod.mjs:36-52 | Destroy on close; timeouts |
| WEB-9 | Med-Low | A mobile-money push result is written to whichever sale is on screen, by row index. Polling continues after the page is left | pos.page.ts:192-223 | Sale token; stop on destroy; server binds request to sale |
| WEB-10 | Low-Med | "Exact" uses `Math.ceil` on a float, so about 6.5% of totals with cents come out 0.01 too high and are refused as `overpaid_non_cash` | pos.page.ts:1144,1206 | Integer minor units |
| WEB-12 | Low | The next user on the PC lands on the previous user's return page | sign-out.ts:373-380 | Same user only |
| WEB-15 | Low (Plausible) | A second confirm dialog resolves the idle warning as "Sign out" | confirm.ts:46-49; idle.ts:270-279 | Dialog queue |
| WEB-16 | Low | The temporary password is kept in history state | sign-in.page.ts:382-384 | In-memory hand-off |
| WEB-17 | Low | `npm install` instead of `npm ci`; Angular CLI analytics id in angular.json | Dockerfile:12; angular.json:6 | `npm ci`; analytics false |

### 3.7 Operations, configuration and engineering (OPS)

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| OPS-1 ✔ | High | Dev default keys only warn: encryption key, files signing secret (forgeable links), `web-url` localhost, files under `target/`. None of them is in deployment.md | api/modules/access/.../SecretBox.java:25-34; notifications/.../SecretSeal.java:23-29; application.yml:78 | Refuse to start; document |
| OPS-2 ✔ | High | The `log` provider is the default and the fallback, and it logs full email and SMS bodies with reset links and codes | notifications/.../LogProviders.java:35,54,73; ProviderRouter.java:332-340 | local/test only; mask; mark FAILED |
| OPS-3 ✔ | High | One URL crashes the web server, and the restart limit is 5 | serve-prod.mjs:55; web/railway.toml:9-10 | Batch 1 |
| OPS-4 | High | No backup, restore or rollback docs or drill. Files sit on a single Railway volume | docs/runbook/deployment.md; docs/prd/pharmacy-prd.md:281 | Backups off Railway; restore drill; S3/R2 |
| OPS-5 | High | Encryption key cannot be rotated (no key id). Changing it locks out every 2FA user and breaks every provider. One key serves both | SecretBox.java:36-53; SecretSeal.java:31-33 | Keyring + re-seal job; separate keys |
| OPS-6 | High | `.railway/railway.ts` has no restart policy; cutover deadline 1 Dec 2026 | api/.railway/railway.ts:7-12; web/.railway/railway.ts:7-12 | Add ALWAYS; cut over and check |
| OPS-7 | High | Job failures are invisible. The subscription job collects per-pharmacy errors in a list and drops them. No metrics registry, no error tracker | tenancy/.../SubscriptionsImpl.java:375-404; Retention.java:72-78 | Job status table, metrics, alerts |
| OPS-8 | Med | No job locking. The deploy overlap (up to 300 s) already runs two instances, giving duplicate SMS and webhooks. Limiters and SSE live in memory, yet deployment.md:59 says to run two or more instances | WebhookRetrySchedule.java:38; RetryJob.java:44-58; NotificationSchedules.java:72 | Advisory lock or ShedLock; `SKIP LOCKED`; fix the doc |
| OPS-9 | Med | MigrationSafetyTest strips DO blocks, so tables forced inside loops (insurance, reporting) are unseen. It has no lock or rewrite rules | architecture-tests/.../MigrationSafetyTest.java:64-67 | Read `relforcerowsecurity` from the database; scan DO |
| OPS-10 | Med | Migrations run with no `lock_timeout`; per-schema Flyway can leave partial state | platform/.../ModuleMigrations.java:51-62 | `lock_timeout 5s`; resume doc |
| OPS-11 | Med | The API holds the DB owner password (migrations run at start), which can turn RLS off | application.yml:53-55 | Migrate-only mode as a pre-deploy step |
| OPS-12 | Med | Swagger on by default; `/actuator/metrics` public; the web proxy forwards both; the API also has a public domain | application.yml:127-142; serve-prod.mjs:12 | Batch 1; remove the API's public domain |
| OPS-14 | Med | No web CI. Deploys upload the folder on disk: live web is an unmerged branch plus a modified angular.json | web/ (no .github); api/.github/workflows/ci.yml | Web CI; deploy from CI on main |
| OPS-15 | Med | Hosted in US-East (`iad`) holding health data; no DPO, breach runbook or processor agreement (also STD-6) | api/.railway/railway.ts:11 | See §4 |
| OPS-16/17 | Low | Unpinned images and actions; no Dependabot, enforcer or CVE scan; `legacy-peer-deps` | Dockerfiles; package.json | Pin; scan |
| OPS-18 | Low | The audit default partition can trap rows for good if the nightly job keeps failing | AuditConfiguration.java:73-76 | Alert; repair function |
| OPS-19 | Low | Clutter (`target-*`, an 8 MB log holding messages); `.env` missing from `.dockerignore`; root email published | api root | Clean up |

### 3.8 Performance and scale (PERF)

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| PERF-1 | High | The branch sale-number row lock is held to commit through the whole checkout, including payment authorization and fiscal issue. With real STK push or VFD (5–30 s), a branch manages a few sales a minute. The Javadoc says otherwise | api/platform/.../DocumentNumbers.java:11,29-30; CheckoutService.java:172; SaleServiceImpl.java:273,298 | External calls outside the transaction; number taken last |
| PERF-2 ✔ | High | Nothing caps total DB demand: virtual threads, `@EnableAsync` with no executor (4 async listeners per sale), 50 pharmacies × 10 slots against a 20-connection pool, no statement or lock timeouts | application.yml:5-20; PlatformConfiguration.java:27 | Bounded executor; global cap; role timeouts |
| PERF-3 ✔ | High | The hashed-asset regex expects `.hash.`, but Angular names files `-HASH`, so everything is `no-cache` with no ETag and no compression. Measured: 714 KB raw against 210 KB gzipped on every load | serve-prod.mjs:67 | Fix the regex; brotli/gzip; API compression |
| PERF-4 | High | Balances, the balance sheet and VAT add up the whole journal on every request (about 750k lines per pharmacy per year) | finance/.../LedgerImpl.java:81-85; FinancialStatementsImpl.java:79-104 | Daily balance read model |
| PERF-5 | Med | Every list runs COUNT + OFFSET. A 50k-row export runs 500 counts | platform/.../Keyset.java:28-34; api/.../Spreadsheet.java:50-66 | Cursor exports; capped totals |
| PERF-6 | Med | uid-sorted lists lack `(tenant_uid[, branch_uid], uid)` indexes on sales, returns, journal, expenses, customers, suppliers, shifts and vouchers | sales/V1__sales_schema.sql:41,125 | Add indexes; EXPLAIN on the QA copy |
| PERF-7 | Med | About 14 audit rows per 3-line sale, copying the append-only ledgers; about 150 M rows over the 7-year window at world scale | EntityAuditListener.java:116-118 | `@AuditIgnore` on child rows; batch inserts |
| PERF-8 | Med | The stock list re-aggregates the whole branch for every page, twice | inventory/.../StockQueriesImpl.java:86-190 | Product stock read model |
| PERF-9 | Med | 4 edge transactions (12–16 round trips) per request before the handler | EdgeFilter.java:137-200 | 30 s cache |
| PERF-10 | Med | Checkout reads settings for every line; 60–80 statements for a 3-line sale; no JDBC batching | CheckoutService.java:213 | Read once; batching |
| PERF-11 | Med | A plan-priced sale loads the whole price list. `GET /price-lists` returns every item of every plan | PricingServiceImpl.java:55-59,353-358,738 | Query by product |
| PERF-13 | Low-Med | `reporting.processed_events` grows for ever | ProjectionsImpl.java | Add to `purge_expired` |
| PERF-14–19 | Low | Journal-number lock per pharmacy; no `sale_lines(batch_uid)` index for recalls; N+1 in purchasing and shifts; statements load full history; both dictionaries in the first chunk; report periods unbounded | see the reviewer's notes | As listed |

### 3.9 Usability and accessibility (UX / A11Y)

The English and Swahili dictionaries have the same 3,178 keys, so nothing is missing in Swahili.

| ID | Sev | Finding | Where | Fix |
|---|---|---|---|---|
| UX-1 ✔ | High | A line with no price in the chosen plan is not blocked: Complete sale stays enabled and the server refuses with a 5 s toast | web/src/app/features/pos/pos.page.ts:300-303; pos.page.html:423 | Block plus an inline list of unpriced lines |
| UX-2 | High | Checkout problems and the "uncertain sale" instruction appear only in toasts of 2.8–5 s that don't pause on hover | core/ui/toast.service.ts:28-47 | Lasting inline alert; dialog for uncertain |
| A11Y-1 | High | 316 labels are not linked to their fields; fields with only a placeholder; unnamed row inputs and icon buttons; no `aria-invalid` | 46 files, e.g. purchasing/receipt-form.page.ts:166-223 | Shared Field component |
| A11Y-2 | High | 17 clickable table rows cannot be opened with the keyboard | sales.page.ts:99, products.page.ts:81, … | A real link in the main cell |
| A11Y-3 | High | Dialogs don't move, trap or restore focus. Escape closes stacked dialogs. Danger confirms focus the destructive button, so a scanner's Enter clears the cart. Focus is lost after the receipt | core/ui/overlay.components.ts:12-35; confirm.ts | Shared Dialog component |
| A11Y-4 | High | Contrast fails: hints 2.56, pills 2.6–3.4, the Complete sale button 3.30, input border 1.21, focus ring 1.29 | web/src/styles.scss:9-37 | The token values the reviewer gave |
| UX-3 | Med-High | 87 of 123 API refusal reasons have no translation, so Swahili users see raw English. Swahili keys drop the product name | core/api/api-error.ts:66-95 | Keys + parameters; CI diff |
| UX-4 | Med-High | Only the POS guards unsaved work. Receipts and orders are lost on navigation or idle sign-out. Save scrolls out of view | app.routes.ts:108 | LeaveGuard for forms; drafts; sticky actions |
| UX-5 | Med | Quantity and discount are clamped silently, and the input can show the rejected value | features/pos/cart.ts:101-113 | Write back + inline message |
| A11Y-5/6 | Med | The picker is not an ARIA combobox; totals are not announced; CONTROLLED and PRESCRIPTION both read "Rx", told apart only by colour | pos.page.html:439-509,606-613 | Combobox pattern; "CD" label |
| UX-6 | Med | The picker always looks focused but is not focused after router navigation or after a modal | pos.page.scss:303-316 | Focus in code |
| UX-7 | Med | Six browser `confirm()` calls remain (against the owner's rule) | webhooks.page.ts:273; account-security.component.ts:450; notifications.page.ts:597; opening-balances.page.ts:340; root-admins.page.ts:183,194 | ConfirmService + a lint rule |
| UX-8 | Med | The suspended page gives no reason, and UNPAID admins cannot reach the subscription page to pay | core/layout/shell.component.ts:113-120 | Kind-specific text; Pay stays reachable |
| UX-9 | Med | Forgot-password has no path for staff logins (`name@CODE`) | features/auth/forgot-password.page.ts | "Ask your manager" |
| UX-10 | Med | A failed list load looks like an empty list, with no retry (43 pages) | core/data/data.ts:83-90 | Error state + Retry |
| UX-11 | Med | No quick "new customer" from the POS | pos.page.html:103-112 | Small modal |
| UX-12 | Med-Low | Several deletes and postings (refund, receipt, adjustment, supplier payment) have no confirm | see reviewer notes | ConfirmService with figures |
| i18n-1 | Med-Low | English on the printed receipt; untranslated tab titles; no plurals ("1 lines") | features/pos/receipt.component.ts:52-58 | TitleStrategy; plural keys |
| UX-13–18 | Low | TZS always shown with 2 decimals; no loading bar; receipt not tuned for 80 mm; user menu keyboard; no skip link; sidebar not remembered; supplier tasks in 3 menu groups | see reviewer notes | As listed |

### 3.10 Standards and regulation (STD)

Items marked *verify* need confirming with counsel, TRA, the Pharmacy Council, TMDA or an accountant.

| ID | Sev | Standard | Finding | Fix |
|---|---|---|---|---|
| STD-1 | High | TRA EFD/VFD *verify* | Fiscal port is `issue(saleNumber,total,tax)`, synchronous inside the checkout lock; one `fiscal_reference`; no status, QR or counters; returns never fiscalised; no tenant TIN/VRN; the receipt prints none | `FiscalDocument` via the outbox, PENDING→ISSUED; credit notes; Z close |
| STD-2 | High | VAT Act 2014; IAS 2.11 *verify* | Only `tax_rate_percent`, so zero-rated and exempt cannot be told apart (medicines exempt?); no input VAT; one 2100 account; output-only VAT report | Tax category; VAT-registered flag + TIN/VRN; input VAT account |
| STD-5 | High | PDPA 2022 | No per-person export, correction or erasure; no consent or lawful basis; diagnosis and prescriber stored as plain fields; customer export not audited | Per-person rights; consent records; retention schedule |
| STD-3 | Med | IAS 2 | Lot cost rather than FIFO or average; no net-realisable-value write-down for expiring stock; `journal_lines` allows debit and credit together | Costing ADR; expiry provision; DB checks |
| STD-6 | Med | PDPA | Registration, cross-border transfer, breach notice, DPO, subprocessor list | Privacy pack |
| STD-7 | Med | Pharmacy Council *verify* | Only prescriber name required; no Rx date or patient; no pharmacist sign-off permission; controlled list capped at 100 | `sales.dispense_rx`; required fields |
| STD-8 | Med | TMDA *verify* | No product registration number; no adverse-drug-reaction reporting | Fields + ADR form |
| STD-9 | Med | OpenAPI 3.1 (PRD §7.6) | No annotations or permissions in the spec; about 165 of 320 routes missing from api-v1.md; field drift | OperationCustomizer from `@Access`; CI diff |
| STD-10 | Med | RFC 9110 ETag | Only products take a version, so other edits overwrite silently | ETag / If-Match (428/412) |
| STD-12 | Med | ISO 8601 / tenant day | UTC day cuts in sales filters, claim batches, FHIR and SMS usage billing; the web uses browser time | TenantCalendar everywhere; ArchUnit rule |
| STD-13 | Med | CloudEvents / Standard Webhooks | Type has no reverse-DNS prefix; signature is over the body only | webhook-id/timestamp/signature |
| STD-17 | Med | ISO 4217 | All money forced to 2 dp HALF_UP, while fx uses HALF_EVEN; 0-dp currencies (UGX, RWF) break | Money type with scale per currency |
| STD-21 | Med | NIST 800-63B-4 | Minimum 8 characters; no breached-password check; Argon2 at 16 MiB | See Batch 2 |
| STD-22 | Med | ISO 27001 A.8.15 | Audit has no IP, user agent, session or outcome; exports not audited | Add fields; audit exports |
| STD-24 | Med | 12-factor; PRD §7.4 | No web lint or CI; no SBOM or dependency scan; secrets not documented | See Batch 6 |
| STD-4, 14–16, 18–20, 23, 25 | Low | various | Invoice prefix restarts numbering; problem-type URIs on a host nobody owns; no WWW-Authenticate or Location; no SemVer; GTIN check digit; case-sensitive batch numbers; phone normalisation; FHIR empty strings; PRD still lists OAuth/Redis as "Must" against ADR 0008 | As listed |

---

## 4. Decisions only the owner can make

These came out of the review but change behaviour, so they need your answer before building:

1. **Approvals.** Which actions need a second person, and above what amounts? Candidates: refunds, stock adjustments and write-offs, destructions, transfers, period unlock, product schedule changes, voucher issue, large discounts and promotions, credit write-offs. Who approves: owner, branch manager, pharmacist?
2. **Default role presets.** Proposal: take `sales.refund`, `inventory.adjust`, `catalogue.manage` and `partners.manage` (credit limits) off the Pharmacist preset, and `finance.manage` (period lock) off Branch Manager. This changes what existing staff can do.
3. **Open shift required to sell (FRD-7).** You hid till shifts (`TILL_SHIFTS_ENABLED=false`). Without shifts, cash can't be tied to a cashier. Re-show shifts, or accept the gap?
4. **Return policy.** A return window (days), refunds only by the original method, and returned Rx/controlled items quarantined rather than put back on the shelf?
5. **Hosting region (PDPA).** Health data is in Railway US-East. Options are an EU or African region, or a Tanzanian host. Counsel should confirm the transfer rules.
6. **Scaling.** Stay single-instance for now (and fix deployment.md), or do job locks, shared limiters and SSE fan-out first?
7. **Fiscal (VFD/EFD) and VAT categories.** Plan this as one project (STD-1, STD-2) before the first VAT-registered client.
8. **TZS display.** Show TZS with 0 decimals on screens and receipts (UX-13)?
9. **Shared counter PCs.** A "shared PC" mode where the session ends when the browser closes, and parked sales expire at end of day (WEB-5)?

---

### Owner's answers (23 Nov 2026)

| # | Decision |
|---|---|
| 1 | **Approvals** apply to all four groups: refunds and returns; stock write-offs (adjustments, destructions, stock-take variances); vouchers, discounts above the limit and promotions deeper than the maximum discount; books and medicine class (period lock moved back, dispensing class changes, credit write-offs). The approver holds a new **Approve** permission (owner, admin and branch manager by default) and is never the person who asked. Approval applies **above a per-pharmacy amount set in Settings (default TZS 50,000)**. The approver types their PIN or password on the same screen. Controlled drugs, period unlock, class changes and credit write-offs always need approval, whatever the amount. |
| 2 | **Role presets:** new pharmacies get the split presets. Existing pharmacies keep their roles, and their admin sees a one-time notice with **"Apply recommended roles"**. No one loses access unless the admin chooses it. |
| 3 | **Till shifts:** shown again. A pharmacy setting **"Require an open shift to sell"** is ON for new pharmacies and OFF for existing ones until their admin turns it on. Closing a shift with a variance above tolerance needs a manager. |
| 4 | **Returns:** a **7-day window** (each pharmacy can change it; after the window a manager approves). Refund **only by the original payment method** (the cash share in cash, the rest to mobile money, card or account). The cashier chooses: back on shelf, damaged, or quarantine. **Prescription and Controlled items never go back on the shelf.** |
| 5 | **Hosting:** move to a **Tanzanian or African host before selling** (leave Railway US-East). Plan the move with backup and restore (ties in with OPS-4). |
| 6 | **Fiscal and VAT:** **one project, after Batches 1–3**, before the first VAT-registered client. It needs TRA VFD sandbox access and an accountant's check of medicine VAT exemptions. |

| 7 | **Scaling:** the system must run as **several instances at once** (for example several ECS tasks). The following must therefore work across instances: background jobs (one runner per job, `SKIP LOCKED` claims), sign-in and rate limits, the per-pharmacy request cap, provider health, live notifications (SSE fan-out) and file storage (S3-compatible, no local volume). Fix deployment.md to match (OPS-8, INT-12, AUTH-14, OPS-4). |

Still open from the list above: 8 (TZS decimals) and 9 (shared-PC mode).

## 5. What held up well

- **Isolation:**
  - forced RLS on every tenant table, including tables created in loops and audit partitions, failing closed
  - `set_config(…, true)` only, so no tenant carries over on pooled connections
  - open-session-in-view off; async listeners and jobs re-enter the tenant scope
  - definer functions have a pinned `search_path`
  - an unknown pharmacy and a non-member get the same answer
- **Ledgers and audit:**
  - append-only triggers and grants on audit, movements, journal, allocations, remittances and price changes
  - every posting balanced, with one posting per source
  - returns are separate credit documents
- **Checkout:**
  - prices come from the server
  - a different plan needs a permission; a missing plan price is blocked
  - the idempotency key takes an advisory lock and has a unique index
  - FEFO with `PESSIMISTIC_WRITE`; no oversell; expired and quarantined batches cannot be sold
  - returns are limited to the quantity sold and priced at the original line's share
  - prescriptions for controlled drugs are enforced
- **Authentication:**
  - 32-byte tokens, stored hashed; Argon2id with a decoy hash, so unknown accounts take as long as known ones
  - TOTP sealed with AES-GCM, constant-time comparison, replay guard
  - no account enumeration
  - sessions revoked on reset and deactivation
  - account routes ignore `X-Tenant-Id`
  - recovery has a 24-hour hold with a cancel link
- **Webhooks:**
  - https only; private and internal addresses refused, checked again on every attempt
  - no redirects; responses discarded; 5 s timeouts
  - retries from 1 min to 24 h, one lane per webhook
- **Files:**
  - signatures bind method, key, expiry and content type
  - only PNG, JPEG, WebP and PDF; `nosniff`
- **Web:**
  - no `[innerHTML]` with user data; the email preview is a sandboxed iframe
  - the return URL stays on the same site
  - the idle limit is enforced by the API
  - zoneless rendering, OnPush, lazy routes
  - no source maps; dependencies current
- **Engineering:**
  - module boundaries enforced by ArchUnit and Modulith
  - MigrationSafetyTest and RowLevelSecurityTest
  - a clock abstraction (`now()` banned), Spotless, JSON logs, readiness checks, graceful shutdown
  - the API image runs as a non-root user
  - no secrets in any git history

## 6. Improvement ideas (not defects)

- **Approvals:** one approval engine and an owner alert digest. These are the largest gain against insider fraud.
- **World checks:** add nightly invariants to pharmacy-world: stock at cost plus units in transit equals ledger 1200; receivables and claims sub-ledgers equal 1100 and 1110; every source document has a journal entry. Add a slow fake payment provider (5–20 s) to the load test to catch PERF-1 and PERF-2.
- **Branch-route test:** one integration test that calls every BRANCH-scoped `{uid}` route as a cashier limited to one branch, against another branch's record.
- **Platform health page for root:** job runs, outbox backlog, provider circuit states, pool usage, size of the audit default partition, and the build SHA.
- **Data export:** an allow-list of columns per table instead of a regex on names.
- **Web components:** one accessible Dialog and one Field component, plus an axe/Playwright accessibility pass in CI.
- **Till:** "Check last sale" after an uncertain checkout (look the sale up by idempotency key); integer minor units throughout the till; a service worker for the app shell.
- **Compliance matrix:** a living document in docs mapping each PRD §7 "Must" to its status, the code that implements it and a test.
