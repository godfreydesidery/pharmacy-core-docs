# Pharmacy Core: QA gaps report

**Run date:** 15 November 2026 (fix status added 16 November 2026, see §0) **Build:** `main` as pushed (web + API) **Method:** full-test skill, visual, headed browser

This report lists every gap found in a full visual test of the system and proposes designs for the capabilities that do not exist yet. Each finding has an ID, what happens, where to see it, and a suggested fix. Screenshots are in the web repo under `docs/.qa/` (git-ignored; paths below are relative to it).

---

## 0. Fix status (16 November 2026)

Everything below was fixed or built after the QA run, in the working trees of `pharmacy-core-api` and `pharmacy-core-web`. **Nothing is committed yet.** The servers run this code on the original local data. The visual check left two open, unposted stock-takes (CNT-000001 and CNT-000002) in the Afya demo pharmacy; they change nothing unless posted.

**Verification:**
- API: full `mvn verify` passes in every module (unit, integration, architecture and formatting checks). The new integration tests are `MoneyScenariosTest`, `ControlledDrugsTest`, `PricingTest`, `TransformationsTest`, `PaymentsTest`, `StockCountsTest`, `StockOperationsTest`, `PlanningTest`, `SubscriptionPlansTest`, `NotificationProvidersTest`, `SecurityFixesTest` and `AdminDataTest`. The row-level-security and module-structure tests pass.
- Web: type check and template compile are clean. 33/33 unit tests pass. Translations: 2,447 keys in English and Swahili, none missing. End to end: the 14 Playwright tests pass. The route sweep now also opens the 18 new pages as admin, cashier and root, and fails on any console error or API 404/405/5xx. A screenshot pass of the new pages is in `docs/.qa/shots/new-features/`.

### 0.1 Findings

| ID | Status | What was done |
|---|---|---|
| C-01 | Fixed | The POS refuses a currency with no rate and resets the row; the server refuses a typed rate more than 25% from the recorded one (`rate_out_of_range`). |
| C-02 | Fixed | Orders and receipts default to the base currency once settings load, cannot save without a rate, and get the same server plausibility check. |
| H-01 | Fixed | Webhook targets that resolve to loopback, private, link-local or metadata addresses are refused, both when saved and when sent. |
| H-02 | Fixed | The method select of a split row shows the real method. |
| H-03 | Fixed | Receipt and order lists show the document currency, with the base total under it. |
| H-04 | Fixed | Platform → Message providers (root) and per-pharmacy provider settings: platform default or own account per channel, encrypted secrets, test send, usage per month. |
| H-05 | Fixed | Inventory → Controlled drugs: per-product register with running balance. A controlled sale requires a complete prescription (`controlled_prescription_incomplete`). Witness PIN and the regulator's report format are still open (§9.6). |
| M-01 | Fixed | No currency is assumed before settings load. |
| M-02 | Fixed | Opening balances wizard (finance), posted once against owner's equity. |
| M-03 | Fixed | Finance → Statements: trial balance, balance sheet, VAT, stock valuation per branch and the cashier report. |
| M-04, M-05 | Fixed | Swahili is complete, including users, shared form messages, product types, payment methods and list footers. |
| M-06 | Fixed | Delivery-log rows open with content and error detail, and can be resent. |
| M-07 | Fixed | Staff have a phone number. |
| M-08 | Fixed | Root admins can edit a pharmacy; the currency stays locked once it has trading. |
| M-09 | Fixed | Root admins can be disabled or removed (never the last one) and have their password reset. |
| M-10 | Fixed | Quick-cash buttons use the notes of the tender's currency. |
| M-11 | Fixed | A supplier invoice number already received from the same supplier is refused. |
| M-12 | Fixed | A tender above 5× the total (and at least 100 over) asks for confirmation. |
| M-13 | Not reproduced | Root visits and changes inside a pharmacy show in that pharmacy's audit trail; covered now by `SecurityFixesTest`. |
| M-14 | Fixed | An admin can unlock a user. Progressive delay or CAPTCHA before the lock is still open. |
| M-15 | Not reproduced | The last accepted time step is stored per user, so a code works once; covered now by `SecurityFixesTest`. |
| M-16 | Fixed | Adding an email that already has an account sends an invitation instead of granting membership. |
| M-17 | Fixed | The phone dashboard fits; the POS has a sticky cart bar with the total and *Complete sale*. |
| L-01 | Fixed | Forms scroll to and focus the first invalid field. |
| L-02 | Fixed | Platform audit shows names. |
| L-03 | Fixed | The message editor flags placeholders the message does not have. |
| L-04 | Fixed | The delivery-log filters are compact, on one toolbar. |
| L-05 | Fixed | Only live sessions are listed; *Sign out other devices*; apps and scripts show as "App or integration (name)". |
| L-06 | Fixed | Dosage forms added. |
| L-07 | Fixed | "Products in stock" counts low-stock products too. |
| L-08 | Fixed | The receiving branch gets a badge for incoming transfers. |
| L-09 | Fixed | The "no rate" warning clears once a rate is typed, and explains where rates are kept. |
| L-10 | Fixed | `/insurance/members/current` returns 200 with no data. |
| L-11 | Fixed | *Complete sale* is disabled while a sale is in flight. |
| L-12 | Fixed | The credit-limit message names the currency. |
| L-13 | Fixed | Handled refusals no longer print as uncaught errors. |
| L-14 | Fixed | A suspended pharmacy sees one page with the reason and next steps. |
| L-15 | Fixed | Accepted invitations are hidden unless *Show all* is on. |
| L-16 | Fixed | Bad links and unknown routes show a "page not found" page. |

### 0.2 Found while fixing

| ID | Severity | Status | Finding |
|---|---|---|---|
| N-01 | High | Fixed | **Stock adjustments never reached the ledger.** Damage, theft, expiry write-offs and count corrections changed stock but not account 1200, so the books overstated inventory. They now post at batch cost to a new account, *5100 Stock Losses & Adjustments*. This covers stock-takes and destructions, which go through the same adjustment. |
| N-02 | Medium | Fixed | **Reports used the server's UTC day.** Between 00:00 and 03:00 in Dar es Salaam, the statements defaulted to yesterday and the cashier report counted those sales on the wrong day. Both now use the pharmacy's time zone. |
| N-03 | Low | Open | Other "today" defaults still use the UTC date: the supplier-payment date, expiry checks, the rate lookup in purchasing and warranty start. These are off by one day only between 00:00 and 03:00 local time. Pass the tenant's zone the same way. |
| N-05 | Low | Fixed | Secondary lines (a code under a name, a date under a batch) ran into the main value on several new pages because the `block` class was only defined inside some components. It is now a global utility. |
| N-04 | Info | Note | `tenancy.subscriptions` and `subscription_invoices` have no row-level security. Like `tenancy.tenants`, they are platform tables: the root admin and the billing run read them across pharmacies, and every query names its pharmacy. |

### 0.3 Designs built (section 9)

| Design | Status | Where |
|---|---|---|
| §9.1 Subscription plans | Built | Platform → Plans; tenant page subscription card; Administration → Subscription. Trial 30 days, invoices 7 days ahead, 7 days' grace, then suspension; payment reactivates. Limits on branches, users, products and SMS; modules for insurance, multi-currency, integrations and advanced reports. |
| §9.2 Pricing | Built | Catalogue → Pricing: price lists per customer type and per customer, quantity breaks, branch prices and promotions. The POS shows where each price came from. |
| §9.3 Repackaging | Built | Inventory → Repack: break bulk (also automatic at the till through pack links), repack into own packs, and compounding from a formula. The output keeps the earliest input expiry and is costed from the inputs. |
| §9.4 Payments | Built | Platform → Payment methods (registry); Administration → Payment methods per pharmacy; the POS lists the pharmacy's methods and channels; *Send to phone* request and confirmation; bank transfer, cheque, card and vouchers (Sales → Vouchers). Split payments across any mix. **The M-Pesa, Mixx, Airtel Money, Selcom and AzamPay adapters are registered but not connected:** they need sandbox credentials from each provider. A simulator is included for testing (disabled by default). |
| §9.5 Notification providers | Built | See H-04. Real delivery is untested: no provider credentials. |
| §9.6 Controlled drugs | Partly built | Register and prescription rule (H-05). Witness PIN, quantity limit per prescription and the regulator's report format are still open. |

### 0.4 Operations from §9.7

| Operation | Status | Where |
|---|---|---|
| Shift / till open and close | Built | POS shift bar (open with a float, close with a cash count); Sales → Shifts shows expected vs counted. |
| Stock-take | Built | Inventory → Stock-takes: a count sheet per batch, printable; posting turns differences into count corrections, valued at cost and journalled (N-01). |
| Reorder suggestions | Built | Purchasing → Reorder suggestions: sales velocity, supplier lead time and cover days, less stock on hand and stock on order. Creates one purchase order per supplier; there is no draft status, so check the quantities before creating. |
| Supplier returns | Built | Purchasing → Supplier returns, with a printable debit note; posts Dr payable / Cr inventory. |
| Batch recall | Built | Inventory → Recalls: every sale of a batch with customer contacts; quarantine blocks sale, transfer and repacking in every branch; release undoes it. |
| Destruction register | Built | Inventory → Destructions, with witness, method, certificate number and a printable certificate. The witness is recorded by name only; there is no second-user PIN. |
| Opening balances | Built | M-02. |
| Period closing | Built | Finance → Period lock. Entries a user dates (expenses, supplier payments, opening balances) are refused on or before the lock date. Sales, receipts and other automatic entries that arrive late are booked on the first open day. |
| Offline POS | Open | Needs its own design: a local queue, sync, and how stock conflicts are resolved. |
| Patient profile and interactions | Open | Allergies, chronic medicines, interaction warnings, dosage labels. |
| Fiscal receipts (TRA EFD/VFD) | Open | Needs TRA registration and a device or VFD account. |
| Data-protection tools | Open | Export or delete a customer's data, consent, retention. |
| Barcode in receiving and stock-take | Open | The POS scan exists; reuse it on these pages. |
| Receipt templates (58/80 mm, A4) | Open | |
| Webhook event `supplier.returned` | Open | The event exists but is not offered to webhooks. |

---

## 1. Summary

The core of the system is sound:
- **Tenant isolation held in every probe.** Cross-tenant ids, header switching, branch restrictions and role permissions held in 19 of 19 checks.
- **The books balance.** Assets $1,557.03 = liabilities $1,927.23 + income $85.20 − expenses $455.40.
- **Stock is safe under concurrency.** Three tills sold the last units at once; one sale went through and two were refused.
- **The main flows work end to end:**
  - order → partial receipt → transfer → sale;
  - prescription → return;
  - credit → payment → write-off → hold;
  - insured sale → claim → batch → approval → remittance;
  - two-step verification, recovery codes and password reset.

The serious problems sit in **multi-currency handling at the edges**, plus one **server-side request forgery** hole. Several capabilities asked for are **not built yet**: subscription plans, price lists, repackaging, payment adapters, a controlled-drugs register, provider settings and shift close.

| Severity | Count | Meaning |
|---|---|---|
| Critical | 2 | Wrong money recorded; must be fixed before any real trading |
| High | 5 | Wrong data shown, security hole, or a regulatory gap |
| Medium | 17 | A feature is incomplete or misleading; workaround exists |
| Low | 16 | Polish, wording, small UX friction |
| Info | 6 | Observations and confirmations |

**Suggested fix order:**
1. The two critical currency bugs: C-01 and C-02.
2. The other high findings: H-01 (SSRF), H-02, H-03 and H-04.
3. Swahili completeness: M-04 and M-05.
4. Then the proposed designs in section 9, starting with subscription plans and payment methods.

---

## 2. Test scope and environment

- **Data.** Run on a **copy** of the local database (`target/local-postgres-qa`), so the demo data stayed untouched.
- **Pharmacies.**
  - **SEED Kilimanjaro Pharmacy:** base **USD**; also accepts TZS and KES; three branches (Moshi Town, Arusha, Moshi Market).
  - **SEED Duka la Dawa:** TZS, one branch; tested while **suspended**.
  - **Afya Pharmacy:** the demo tenant, used for isolation checks.
- **Roles exercised.** Root admin, second root admin, pharmacy admin, branch manager (all branches), cashier (one branch), custom "Dispenser" role (one branch), invited pharmacist, and one person who belongs to two pharmacies.
- **Records created.**
  - catalogue: 16 products covering every product type and tracking mode, 3 manufacturers, 1 category;
  - parties: 3 suppliers, 69 customers;
  - purchasing and stock: 1 PO, 3 goods receipts (one in TZS), opening stock for 8 products, 2 adjustments, 1 transfer;
  - sales: 14 sales, 2 returns;
  - money: expenses, exchange rates, a supplier payment;
  - insurance: insurer, scheme, member, claim, batch, remittance.
- **Viewports.** Maximised desktop, tablet (768 px) and phone (390 px); light and dark themes; English and Swahili.

---

## 3. Critical

### C-01 POS shows a TZS payment but records it as USD at rate 1 when no rate exists
**Area:** POS / multi-currency **Layer:** frontend + backend

**What happens.** In a USD pharmacy with no TZS rate recorded yet:
1. The cashier sets a payment row to *Cash / TZS / 20000*.
2. A toast says "No TZS rate yet", but the row keeps showing TZS.
3. The sale completes as **KILI-MSH-000008**, storing *CASH USD 20,000 at rate 1*.
4. The till shows **"Paid $20,020 · Change $19,999"** on a **$21** sale, and the receipt prints the same.

**Fix:**
- When a currency has no rate, disable it in the payment row, or require a rate typed inline.
- Keep the select bound to the payment's actual currency.
- The API should refuse a foreign payment without a valid rate.

See also M-12, the tender sanity check.

**Evidence:** `shots/pos/015-mixed-currency-1.png`, `016-mixed-currency-2.png`, `017-receipt-mixed.png`

### C-02 Purchase orders saved in TZS at a locked rate of 1 in a USD pharmacy
**Area:** purchasing / multi-currency **Layer:** frontend (cause) + backend (no guard)

**What happens.** Opening *Purchase orders → New* directly and entering USD costs produced **PO-MSH-000001: "Order total TZS 880.00 … In USD at the locked rate 1 per TZS"**. The receipt against it inherited the rate. Payables, stock value and later FX differences are therefore all wrong.

**Root cause:**
- `order-form.page.ts` and `receipt-form.page.ts` set `currency = signal(this.base())` once, when the form is built.
- `SettingsStore.currency` falls back to `'TZS'` until the settings request returns.
- So a direct page load starts in TZS, with `rate = signal(1)`.
- The API accepts any positive rate without comparing it with the rate table.

**Fix:**
- Derive the default currency reactively, or wait for settings to load.
- Default the rate from the rate table.
- Remove the TZS fallback.
- Have the API refuse or flag a rate far from the recorded one.

**Evidence:** `shots/purchasing-inventory/009-po-after-partial.png`, `010-grn-over-receipt.png`

---

## 4. High

| ID | Finding | Where / evidence | Suggested fix |
|---|---|---|---|
| H-01 | **Webhooks can target the server's own internal addresses (SSRF).** A pharmacy admin registered `http://localhost:8080/actuator` and `http://127.0.0.1:54330` (the database port); both were accepted. Only `169.254.169.254` and non-HTTP schemes were refused. | `isolation-results.json` | Refuse loopback, private, link-local and internal host names after DNS resolution; check again at delivery time. |
| H-02 | **Split payment row shows "Cash" while it is mobile money.** After *+ Split*, the new row's select displays *Cash*, but it has a Reference box and was recorded as "Mobile money · MAN-a18242cd". Cash-up will not reconcile. | `shots/pos/016-mixed-currency-2.png`, `017-receipt-mixed.png` | Bind the select to the row's method; give new rows an explicit default. |
| H-03 | **Goods receipts list prints foreign totals with the base symbol.** TZS 2,990,000 receipts show as "$2,990,000.00" in a USD pharmacy. | `shots/purchasing-inventory/009-b-purchasing-receipts.png` | Format with the receipt currency, and add the base equivalent. |
| H-04 | **Notification providers can only be set by server environment variables.** SMTP, Africa's Talking, Twilio, Beem and WhatsApp are chosen with `PHARMACY_*_PROVIDER` variables (local default `log`). There is no screen for providers or credentials, no per-pharmacy sender, no test message, and no balance or health view. | `shots/notifications/001-templates.png` | Design in §9.5. |
| H-05 | **Controlled drugs are handled like ordinary prescription medicines.** Morphine (schedule CONTROLLED) gets only the prescription form. There is no register with a running balance, no witness, no stricter role, and no register report. | `shots/pos/011-controlled-after-complete.png` | Design in §9.6. |

---

## 5. Medium

| ID | Finding | Evidence | Suggested fix |
|---|---|---|---|
| M-01 | **Money formats in TZS until settings load.** `SettingsStore.currency` defaults to "TZS", so a USD/KES pharmacy briefly shows TZS, and forms built in that window inherit it (cause of C-02). | — | No currency until settings load; show a skeleton. |
| M-02 | **No opening balances or owner capital; cash goes negative silently.** Kilimanjaro shows Cash −$1,471.03 and Mobile Money −$12. | `shots/finance-reports/012-chart-of-accounts.png` | Opening-balance wizard (cash, bank, mobile, stock, payables, receivables, capital); warn before a payment exceeds the account balance. |
| M-03 | **Core financial and regulatory reports missing.** Reports has Sales, Products, Profit & loss and Payments only. | `shots/finance-reports/014…017` | Add balance sheet, trial balance, cash flow, VAT/TRA, stock valuation, write-offs, controlled-drug register, cashier/shift report, purchases by supplier, claims ageing, period lock. |
| M-04 | **English left in Swahili on 30 of 40 pages.** Worst: product types and tracking modes, POS payment methods, the "N shown" footer on every list, the users page, the account page and the product form. | `presentation` shots; list in `state.json → swahiliLeftovers` | Move the remaining literals into the dictionaries; translate the `PRODUCT_TYPES`/`TRACKING_MODES`/payment-method labels and the pager. |
| M-05 | **Users page and shared form messages are hard-coded English.** "Users", "Save", "Can work in every branch"; `controlError` returns "Required", "At least 8 characters". | `users.page.ts`, `core/ui/forms.ts` | Same as M-04. |
| M-06 | **Delivery log rows cannot be opened.** No message content, no error detail, no resend; an admin cannot help someone whose email never arrived. | `shots/notifications/002-kili-invite-email.png` | Row detail drawer with the rendered message (secrets masked), provider response, retry/resend. |
| M-07 | **Staff have no phone number,** so SMS/WhatsApp alerts can never reach them, although the preference grid offers those channels. | `shots/notifications/010-account.png` | Profile section (name, phone, photo) on the account page and user form. |
| M-08 | **Root admins cannot edit a pharmacy after registering it.** | `shots/platform/009-detail-kili.png` | Edit name, legal name, contacts, time zone; lock currency once trading. |
| M-09 | **Root administrators cannot be removed or disabled,** and there is no password reset for another root admin. | `shots/platform/015-root-admins-after.png` | Deactivate/remove (not the last one), reset, require two-step for root admins. |
| M-10 | **Quick-cash buttons use shilling notes in every currency** (1,000 / 2,000 / 5,000 / 10,000 in a USD pharmacy). | `qa-results/06-pos…/test-failed-1.png` | Denominations per currency (USD 1/5/10/20/50/100; KES 50…1000), following the row currency. |
| M-11 | **The same supplier invoice can be received twice** (SEED-KD-7781, twice). | `shots/purchasing-inventory/009-b-purchasing-receipts.png` | Warn/refuse a duplicate supplier + invoice number. |
| M-12 | **No sanity check on a tender far above the total.** A $20,000 tender on $21 gives $19,999 change with no confirmation. | `shots/pos/016-mixed-currency-2.png` | Confirm above e.g. 5× the total or the largest note. |
| M-13 | **Root admin access is invisible to the pharmacy.** Root reads inside Kilimanjaro are not in its audit log. | `isolation-results.json` | Log root actions in the tenant's audit; optionally require a reason and time-boxed access. |
| M-14 | **Anyone can lock a user out, and only time clears it.** After 4 wrong passwords: 423 for the lock period, even with the right one. No admin unlock, no "locked" marker, no CAPTCHA/IP throttling first. | `isolation-results.json` | Admin unlock + locked badge; progressive delay or CAPTCHA before locking; per-IP limits. |
| M-15 | **An authenticator code works twice within its 30 seconds.** | `shots/auth/` | Remember the last accepted time step per user (RFC 6238 §5.2). |
| M-16 | **Adding an existing account from another pharmacy grants membership silently.** Kilimanjaro "created" `admin@afya.co.tz`: membership added at once, no consent or notice. The form says the password is ignored. Also reveals the email exists. | `shots/tenant-setup/*user-from-other-pharmacy.png` | For an existing email, send an invitation the person accepts. |
| M-17 | **Phone layouts:** the dashboard is 256 px wider than a 390 px screen (page zooms out). The POS puts the cart and *Complete sale* below the whole product list. | `shots/presentation/019-phone-dashboard.png`, `020-phone-pos.png` | Fix the dashboard grid; add a sticky cart bar on phone/tablet. |

---

## 6. Low

| ID | Finding | Evidence |
|---|---|---|
| L-01 | Forms do not scroll to the first invalid field on submit (the weak-password error is hidden below the fold). | `shots/platform/007-register-weak-password.png` |
| L-02 | Platform audit shows record ids ("Tenant 8e7857b6"), not names. | `shots/platform/016-platform-audit.png` |
| L-03 | Message editor accepts placeholders a message does not have (`{{name}}` in Low stock renders blank). | `shots/notifications/003-template-edited.png` |
| L-04 | Delivery-log filters stretch full width and stack; stray scroll arrows beside the tabs. | `shots/notifications/006-deliveries.png` |
| L-05 | Signed-in device list grows without bound; API clients show as "Browser on unknown system". | `shots/notifications/010-account.png` |
| L-06 | Dosage forms missing: oral solution, spray, patch, powder for reconstitution, lotion, pessary, lozenge, infusion. | — |
| L-07 | "Products in stock" card excludes low-stock items (Arusha shows 0 with 2 products on hand). | `shots/purchasing-inventory/006-b-arusha-stock.png` |
| L-08 | A receiving branch gets no badge or alert for an incoming transfer (page opens on *Outgoing*). | `shots/purchasing-inventory/004-b-transfers-at-arusha.png` |
| L-09 | Goods receipt keeps saying "No TZS rate" after a rate is typed inline; it should offer to save the rate. | `shots/purchasing-inventory/002-b-grn-expired-attempt.png` |
| L-10 | POS logs a 404 for every customer without insurance (`/insurance/members/current`). | console |
| L-11 | Double-clicking *Complete sale* shows "This conflicts with an existing record" after a successful sale (the sale is correctly created once). | `shots/pos/019-double-click.png` |
| L-12 | Credit-limit message has no currency ("…to 628.00, over the credit limit of 500.00"). | `shots/pos/*b-credit-over-limit-result.png` |
| L-13 | Handled API refusals are printed as uncaught `ERROR ApiError` stack traces in the console. | console |
| L-14 | A suspended pharmacy's users can still open every page, each failing with 403; show one "suspended" page with contact/payment steps. | `shots/robustness/*suspended-after-pick.png` |
| L-15 | Accepted invitations stay under "people who have not joined yet". | `shots/presentation/005-sw-admin-users.png` |
| L-16 | Bad links show "uid is not a valid value."; unknown routes silently go to the dashboard (no "page not found"). | `shots/presentation/028…029` |

---

## 7. UI/UX review (collected)

**What is good:**
- **Consistent look.** Cards, chips, tables and toasts look the same across the app.
- **Dark mode** is clean.
- **Clear empty states,** such as "No sales in this period".
- **Helpful inline hints:**
  - margin and below-cost warning;
  - "earliest-expiring batches are sold first";
  - expiry colour codes;
  - credit "Owes / Available" on the till.
- **Accounts:**
  - the statement reads like a real one;
  - the claim lifecycle is readable (Draft → Submitted → Approved → Paid).
- **Mistakes are caught:**
  - the prescription form opens automatically;
  - serial picker for equipment;
  - over-stock capped at what is on hand;
  - expired batches refused on receipt;
  - over-receipt refused against an order.

**What to improve (beyond the findings above):**
1. **Consistency of language.** Swahili coverage (M-04/M-05) and jargon: "tenant" in messages ("already a member of this tenant"), raw codes such as `stock.low` under template names.
2. **Feedback timing.** Toasts linger and pile up (three at once in several screens). Errors appear as toasts far from the field that caused them (L-01).
3. **Phone and tablet selling** (M-17): a sticky cart and larger tap targets for the till.
4. **First-run guidance.** A new pharmacy lands on an empty dashboard. Add a setup checklist: settings, branches, users, products, opening stock, opening balances, exchange rates, providers.
5. **New branches** do not appear in the branch switcher until the workspace is chosen again (the toast says so; it should refresh itself).
6. **Lists:** filters vary between pages (inline chips vs full-width selects); the "N shown" pager is English-only.
7. **Accessibility:** keyboard focus is visible on sign-in. Still to check: screen-reader labels on icon-only buttons and colour contrast of muted text in dark mode.

---

## 8. Multi-tenancy and security results

| Check | Result |
|---|---|
| Kilimanjaro admin sends `X-Tenant-Id` of Afya | 403 `forbidden` ✓ |
| Afya sale / product / customer / account read by id inside Kilimanjaro | 404 ✓ |
| Afya branch id as `X-Branch-Id` in Kilimanjaro | 403 `branch_not_allowed` ✓ |
| Sell or edit an Afya product from Kilimanjaro | 422 / 404 ✓ |
| Moshi-only cashier reads or sells at Arusha | 403 `branch_not_allowed` ✓ |
| Cashier on users, settings, journal, platform | 403 ✓ |
| Tenant admin on platform tenants | 403 ✓ |
| Suspended pharmacy (sign-in allowed, data refused) | 403 `tenant_suspended` ✓ |
| No token / bogus token / token after sign-out | 401 ✓ |
| Lockout after repeated wrong passwords | 423 after 4 ✓ (see M-14) |
| Webhook to loopback / database port | **accepted ✗ (H-01)** |
| Webhook to cloud metadata / ftp | 422 ✓ |
| Root admin activity visible in the tenant audit | **no ✗ (M-13)** |
| Document numbers per tenant (Afya `INV-HQ-…`, Kilimanjaro `KILI-MSH-…`) | ✓ |
| Three parallel sales of the last 50 units (30 each) | 1 sold, 2 refused `insufficient_stock` ✓ |
| Password-reset link reused | refused ✓ |
| Forgot-password for an unknown email | same answer as a known email ✓ |
| Wrong two-step code; recovery code | refused; accepted ✓ |
| Same two-step code twice | **accepted ✗ (M-15)** |

---

## 9. Proposed designs (decisions taken with the product owner)

### 9.1 Subscription plans
The owner chose: **limits, feature modules, trial and grace period, billing and invoices**.

- **Plans, as root-admin data, not code.**
  - Example tiers: **Starter** (1 branch, 3 users, 500 products, 200 SMS/month), **Standard** (3 branches, 15 users, unlimited products, 1,000 SMS, insurance), **Pro** (unlimited branches, multi-currency, integrations/webhooks, advanced reports, API access).
  - Prices are monthly or yearly, per plan, in TZS/USD.
- **Entitlements.** Each tenant has a `subscription`: plan, status (TRIAL, ACTIVE, PAST_DUE, SUSPENDED, CANCELLED), period end, add-ons such as extra branches or SMS packs.
  - The API enforces limits on create (branch, user, product) with a `plan_limit_reached` problem.
  - Modules are gated by an entitlement check in `@Access` (e.g. `module = INSURANCE`).
  - The UI hides gated menus and shows an upgrade prompt.
- **Trial and grace.**
  - A 14- or 30-day trial on registration.
  - Reminders at 7, 3 and 1 days before renewal.
  - When unpaid: PAST_DUE with a 7-day grace period (banner, full use), then automatic suspension. Data is kept and reactivates on payment.
  - Today's manual suspension becomes one of these statuses.
- **Billing.**
  - Invoices per period, with numbers, VAT and a PDF.
  - Payment by mobile money or card through the payment adapters in §9.4, with automatic renewal when a token or standing order exists.
  - Root admin sees MRR, overdue tenants, and usage per tenant (users, branches, sales, SMS used). This closes the "tenant page shows no usage or plan" gap.
- **Self-signup** (optional): a public "start a trial" form that creates a tenant in TRIAL, with email verification.

### 9.2 Pricing
The owner chose: **price lists per customer type, per-customer special prices, quantity breaks, branch prices and promotions**.

- **Customer types become tenant data, not an enum.** Defaults: Retail, Wholesale, Insurance, Corporate, Staff, NGO/Government. Today only Retail/Wholesale/Insurance exist and "Corporate" is not offered.
- **Price lists.**
  - A list has a name, currency and validity dates.
  - Each line is a fixed price or a % off/on the retail price, per product or per category.
  - A customer type has a default list; a customer may have their own list.
- **Price resolution at the till.** The most specific rule wins:
  1. customer special price;
  2. customer's price list;
  3. customer-type list;
  4. branch price;
  5. product retail price.

  Quantity breaks apply within the chosen list. Promotions (date range, branch, product or category, % or fixed, optional "buy X get Y") apply last, capped by the maximum discount.
- **Transparency.** The POS line shows the list or promotion applied; receipts and sales reports keep the list price and the discount given. A margin guard warns below cost.
- **History.** Every price change is recorded (who, when, old → new), with a price-change report.

### 9.3 Repackaging and compounding
The owner chose: **break bulk, repack into own packs, compounding**.

- **Units and conversions per product.** A base unit (tablet), pack units (strip = 10, box = 100, tin = 1,000) and the units it may be bought or sold in. Stock is held in the base unit; prices per unit; the POS lets the cashier pick the unit.
- **Break bulk.** Selling a strip from a box converts automatically, with no separate step. The batch and expiry are kept.
- **Repack job:**
  - The source batch(es) and quantity go into a new pack product. Example: 1,000 tablets → 100 "Paracetamol 10s (own pack)".
  - The pack gets its own batch number and inherits the **earliest source expiry**, or a shorter in-use expiry.
  - Losses are recorded, cost rolls up, and a label is printed: name, strength, batch, expiry, pharmacy, directions.
- **Compounding:**
  - A formula (bill of materials) lists the ingredients and quantities for a preparation.
  - A compounding job consumes the ingredient batches and produces a batch of the preparation, with a beyond-use date, a costing and the pharmacist's sign-off.
  - Each job is recorded in a compounding register.
- **Traceability.** From any sold pack or preparation you can trace back to the source batches, which recalls need.

### 9.4 Payment methods, adapters and split payments
The owner chose:
- **mobile money:** M-Pesa, Mixx by Yas (Tigo Pesa), Airtel Money, and an aggregator (Selcom/AzamPay or similar);
- **other methods:** bank transfer/deposit, card terminal, cheque, vouchers/gift cards.

**Today:** methods are a fixed enum (cash, card, mobile money, insurance, credit, account credit). The POS does not offer bank transfer or cheque, although the receivables form does. Mobile money gets a generated `MAN-…` reference from a "manual" provider. Split payments exist and work, but see H-02.

**Proposal:**
- **Platform payment-method registry** (root admin). Each method type has:
  - a code and a kind: CASH, MOBILE_MONEY, BANK, CARD, CHEQUE, VOUCHER, CREDIT, INSURANCE;
  - the required fields: reference, phone, bank and account, cheque number and date;
  - the ledger account it posts to;
  - an optional **adapter**.
- **Adapters** implement one port: `initiate(amount, currency, payer)` → pending, then callback/poll → confirmed or failed, plus refund. Examples: M-Pesa STK push, Airtel/Mixx push, aggregator checkout, card-terminal reference capture.
  - Platform credentials are shared; a pharmacy may connect its own merchant account (paybill/till, aggregator keys).
- **Per-pharmacy enablement.** Which methods each pharmacy and branch accepts, in which currencies, with which own merchant accounts.
- **Split payments** stay: any number of rows of any enabled method and currency. Each row has its own status, and a sale completes only when all electronic rows are confirmed. Push payments show a "waiting for customer" state with a timeout.
- **Reconciliation.**
  - A daily settlement import or API pull per adapter, matched to payments.
  - Unmatched items are listed.
  - Bank deposits and cheques have a "cleared" step that moves money from *undeposited funds* to the bank account.
- **Vouchers/gift cards:**
  - sold or issued with a number, balance and expiry;
  - redeemed as a tender row;
  - tracked as a liability account.

### 9.5 Notification providers
The owner chose: **platform default + each pharmacy's own**.

- **Root admin screen** for the platform providers: SMTP or email API, Africa's Talking, Twilio, Beem and WhatsApp Business. It covers credentials (stored encrypted), sender ID or from-address, a test send, and the balance or health of each.
- **Pharmacy settings** to use the platform default or connect its own account per channel (own SMS sender ID, own email domain); test send included.
- **Usage metering** per pharmacy per channel. This feeds the plan's SMS allowance (§9.1) and any overage billing.
- **Delivery log detail and resend** (M-06), and staff phone numbers (M-07).

### 9.6 Controlled-drugs register (compliance)
- A per-branch register for each controlled product: every receipt, sale, return, transfer, adjustment and destruction is a register line, with date, document, patient, prescriber and registration, quantity, running balance and user.
- **Sale rules:**
  - restricted to a "controlled drugs" permission;
  - a second-person witness (PIN) for dispensing and destruction;
  - quantity per prescription limited;
  - the prescription image is mandatory.
- **Reports:** a register report in the regulator's format, and a balance check against physical count.

### 9.7 Other operations not yet built
Each needs its own design; listed for planning:
- **Shift / till open and close:** float, cash count, expected vs counted, Z-report, a hand-over between cashiers.
- **Offline POS:** queue sales locally and sync later, with stock conflicts resolved at sync.
- **Stock-take:** count sheets per location and variance posting.
- **Reorder suggestions** from sales velocity and lead time, generating draft purchase orders.
- **Supplier returns** (return to supplier) with debit notes.
- **Expired/damaged destruction register** with witness and certificate.
- **Batch recall:** find every sale of a batch, contact customers, quarantine stock.
- **Patient profile:** allergies, chronic medicines, dispensing history and interaction warnings; dosage label printing.
- **Fiscal receipts** through TRA EFD/VFD.
- **Data-protection tools:** export or delete a customer's data, consent, retention.
- **Opening balances wizard** (M-02).
- **Period closing** (lock dates).
- **Barcode scanning** in receiving and stock-take, not only the POS.
- **Per-pharmacy receipt and print templates:** thermal 58/80 mm and A4.

---

## 10. Pharmacy operations coverage

| Operation | Status | Notes |
|---|---|---|
| Tenant onboarding, suspension, root admins | Exists | Edit tenant and root-admin management added (M-08, M-09) |
| Branches, roles, users, invitations | Exists | Existing accounts are invited (M-16 fixed) |
| Catalogue: medicines, equipment, supplies, care, supplements, baby | Exists | Dosage forms added (L-06) |
| Purchase orders → partial receipt, over-receipt refused | Exists | Currency defect (C-02) |
| Direct receipt, foreign currency, expired batch refused | Exists | Duplicate invoice (M-11), list currency (H-03) |
| Opening stock, adjustments, below-zero refused | Exists | — |
| Transfers between branches | Exists | No incoming alert (L-08) |
| Expiry tracking (FEFO), alerts | Exists | — |
| POS: cash, change, mobile, card, credit, account credit, split | Exists | C-01, H-02, M-10, M-12 |
| Prescription capture | Exists | — |
| Controlled-drug register | Built | Witness PIN, quantity limit, regulator format open (§9.6) |
| Serial-number equipment sale and return | Exists | — |
| Returns (cash, store credit), quantity capped | Exists | — |
| Discounts with a maximum | Exists | Promotions missing (§9.2) |
| Price lists, customer-type and special prices, quantity breaks | Built | §9.2 |
| Repackaging, break bulk, compounding | Built | §9.3 |
| Customer credit: limit, hold, payment, write-off, statement | Exists | — |
| Insurance: payer, scheme, member, co-pay, claim, batch, approval, remittance | Exists | — |
| Expenses, supplier payments, exchange rates, journal, chart of accounts | Exists | Opening balances and period lock added; adjustments now journalled (N-01) |
| Reports | Exists | Statements, VAT, stock valuation, cashier report (M-03) |
| Notifications: templates, preferences, inbox, delivery log | Exists | Providers, detail and resend, staff phones added |
| Payment adapters and method registry | Built | Mobile-money adapters need provider credentials (§9.4) |
| Subscription plans and billing | Built | §9.1 |
| Shift / till close, stock-take, recalls, supplier returns, destructions, reorder | Built | §0.4 |
| Offline POS, patient profile, fiscal receipts, data protection, receipt templates | **Missing** | §0.4 |
| Two-step, passkeys, sessions, lockout, password reset | Exists | Admin unlock added; passkeys not tested |
| English / Swahili | Exists | Complete (M-04, M-05) |

---

## 11. Not tested or not verified

- **Passkeys (WebAuthn).** Needs a virtual or real authenticator.
- **Real delivery through SMTP/SMS/WhatsApp, and real mobile-money payments.** No provider credentials; only the log provider and the payment simulator were exercised.
- **Webhook delivery to a real endpoint.** I did not send data to external services.
- **Receipt printing on thermal and A4.** `window.print` cannot be driven; check by hand.
- **Time zone edge.** Reproduced while fixing (N-02) and fixed for reports; other defaults remain (N-03).
- **Load and volume beyond 70 records per list.** Paging and search at 69 customers worked.
- **The Docker image running against a real PostgreSQL.** CI builds the image; it has not been run.

---

## 12. Reproducing the run

- **Harness** (git-ignored), in `pharmacy-core-web/docs/.qa/`:
  - `qa.config.ts` and `specs/01…12`: Playwright, headed, maximised;
  - `isolation.py`: API isolation and security probes;
  - `findings.jsonl`: raw findings;
  - `state.json`: created ids and passwords;
  - `shots/`: screenshots.
- **Run:** `npx playwright test -c docs/.qa/qa.config.ts <spec>` with the API and the web app on 8080/4300. The QA copy of the database is `application/pharmacy-core/target/local-postgres-qa`; start it with `PHARMACY_LOCAL_DATA=target/local-postgres-qa`.
- **Test logins** (QA copy only):

  | Login | Password |
  |---|---|
  | `seed.admin@kili.test` | `Kili#2026qa` |
  | `seed.manager@kili.test` | `Mgr#2026new` |
  | `seed.cashier@kili.test` | `Cash#2026qa` (two-step on) |
  | `seed.admin@duka.test` | `Duka#2026qa` (suspended) |
