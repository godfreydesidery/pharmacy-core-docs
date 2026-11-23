# Records of processing

What personal data Pharmacy Core holds, about whom, why, on what lawful basis and for how long.
It is written from the database migrations and code of `pharmacy-core-api` (main, 23 Nov 2026).
Table names are given so each entry can be checked against the code.

Every pharmacy's rows carry `tenant_uid`, and PostgreSQL row-level security is **forced** on them.
The application role cannot bypass it, so one pharmacy never sees another's data.

"Controller" below means who decides the purpose (see [README](README.md), C1). Lawful bases are
**proposals for counsel**. Health data is sensitive personal data under the Act, so its basis is
question C3. Retention details are in [retention-schedule.md](retention-schedule.md).

## 1. Customers and patients (controller: the pharmacy)

| | |
|---|---|
| **Who** | People who buy from a pharmacy, have a credit account, or are dispensed medicine. A patient may be someone other than the buyer (a child, a relative) |
| **Data** | `partners.customers`: name, phone, email, address, customer type (retail, wholesale, insurance), insurance provider and number, credit limit. `sales.sales`: customer name, **patient name**, and the medicines bought (`sales.sale_lines`: product, quantity, price, schedule OTC / PRESCRIPTION / CONTROLLED, batch). `receivables.*`: credit balance, payments, allocations. `sales.vouchers`. `integrations.payment_intents`: the phone number for a mobile-money payment |
| **Health data** | Yes. What a named person was dispensed reveals their health, especially prescription-only and controlled medicines. The controlled-drugs register is built from these sales |
| **Purpose** | Selling and dispensing, receipts, credit accounts, insurance claims, legally required registers (controlled drugs), stock and accounting |
| **Lawful basis (proposed)** | Contract with the customer (sale, credit). Legal obligation for registers and tax records. For health data: provision of medical care under a health professional (the pharmacist), or written consent. **Counsel C3** |
| **Source** | Staff enter it at the till or in the customer list; imports (Excel/CSV) |
| **Retention** | While the pharmacy uses the platform; deleted at the pharmacy's erasure. No per-person deletion yet (Batch 9) |
| **Who can see it** | The pharmacy's staff by permission; the bulk export only with `data.export` (administrators, branch managers, accountants) |

## 2. Prescriptions and prescribers (controller: the pharmacy)

| | |
|---|---|
| **Who** | Patients (above) and the prescribing doctors or clinicians |
| **Data** | On each prescription sale (`sales.sales`): `prescriber_name`, `prescriber_registration` (professional registration number), `prescription_reference`, `patient_name`. Prescription images uploaded as files with purpose `PRESCRIPTION` (`files.files`, bytes in S3) |
| **Health data** | Yes (the prescription itself) |
| **Purpose** | Legal dispensing of prescription-only and controlled medicines; evidence for inspections; insurance claims |
| **Lawful basis (proposed)** | Legal obligation (medicines and pharmacy law); medical care under a health professional. Prescribers: legal obligation / legitimate interest (professional data). **Counsel C3, C8** |
| **Retention** | As customers. Sector law may set a **minimum** retention for prescriptions and registers (**Counsel C8**) |

## 3. Insurance members and claims (controller: the pharmacy)

| | |
|---|---|
| **Who** | Insured customers and patients, and the principal member of a family cover |
| **Data** | `insurance.members`: customer name, member number, principal's name, scheme, validity dates. `insurance.claims`: member number, **patient name**, **diagnosis code**, pre-authorisation number, the sale, amounts. `insurance.claim_lines`: products, quantities, rejection reasons. `insurance.claim_batches`, `insurance.remittances` |
| **Health data** | Yes: diagnosis codes and the medicines claimed |
| **Purpose** | Billing insurers (payers) and reconciling what they pay |
| **Lawful basis (proposed)** | Contract (the member's cover, the pharmacy's agreement with the payer); for health data, consent or medical care (**Counsel C3**) |
| **Disclosed to** | The insurer (payer), by the pharmacy, outside the platform (claim files) |
| **Retention** | As customers; payers' contracts and sector law may set minimums (**Counsel C8**) |

## 4. Pharmacy staff (controller: the pharmacy; the platform for personal accounts, C1)

| | |
|---|---|
| **Who** | Owners, pharmacists, cashiers, store keepers, accountants |
| **Data** | `access.users`: name, sign-in email, phone, contact email, password **hash** (never the password), TOTP secret (sealed with the encryption key), passkeys (public keys), recovery codes (hashed). `access.memberships`: role in each pharmacy, branches, staff number, job title, work email and phone, last active time, and the `former_*` snapshot kept when someone leaves. `access.sessions`: sign-in time, **IP address**, **user agent**. Sign-in challenges, password resets, phone codes, email changes, invitations. The **audit trail** (`audit.entries`): who did what and when, with before/after values. Shifts and the cashier's name on sales |
| **Purpose** | Access control and security, accountability (who sold, who approved), shift cash-up, fraud prevention |
| **Lawful basis (proposed)** | Contract (employment, the service); legitimate interest in security and fraud prevention; legal obligation (records) |
| **Retention** | Sessions 30 days after expiry; codes 7 days; invitations 90 days; audit 7 years by default (2-year floor); accounts while active (**Counsel C9**) |

## 5. Suppliers' and insurers' contact people (controller: the pharmacy)

| | |
|---|---|
| **Data** | `partners.suppliers`: name, contact person, phone, email, TIN, address. `insurance.payers`: contact, email |
| **Purpose** | Purchasing, payments, claims |
| **Lawful basis (proposed)** | Contract; legitimate interest (business contacts) |
| **Retention** | While the pharmacy uses the platform |

## 6. Messages to people (controller: the pharmacy, or the platform for account messages)

| | |
|---|---|
| **Data** | `notifications.messages`: channel (email, SMS, WhatsApp, in-app), recipient address, subject and body. Account messages (reset links, invitations, sign-in codes) **keep no text** once sent (`keep-account-messages: false`). `notifications.inbox` (in-app notices), `notifications.preferences` |
| **Purpose** | Receipts by SMS (if the pharmacy turns it on), digests, account security messages |
| **Disclosed to** | The email/SMS/WhatsApp provider that delivers it ([subprocessors.md](subprocessors.md)) |
| **Retention** | Sent or failed messages 180 days; read in-app notices 180 days after reading |

## 7. Pharmacies as customers of the platform (controller: the platform)

| | |
|---|---|
| **Data** | `tenancy.tenants`: name, legal name, country, contact name, contact email, contact phone, address. `tenancy.branches`: address, phone, licence number. `tenancy.subscriptions`, `tenancy.subscription_invoices` |
| **Purpose** | Running the account, billing, support, legal and tax records |
| **Lawful basis (proposed)** | Contract; legal obligation (tax) |
| **Retention** | While the account exists. After erasure, the tenant row stays as a **tombstone without its contact details**, and the billing records stay as the platform's tax records (**Counsel C8**) |

## 8. Technical and security data (controller: the platform)

| | |
|---|---|
| **Data** | Application logs in CloudWatch, with a correlation id. The message providers' log lines carry only a masked address and the message length, never the body. Load balancer and WAF metrics. Database logs: slow statements (**the SQL text, which can include values**) and schema changes. Webhook deliveries (`integrations.deliveries`, payload text) to the pharmacy's own systems. Idempotency keys. Internal event log |
| **Purpose** | Running, securing and debugging the service |
| **Lawful basis (proposed)** | Legitimate interest (security, service operation) |
| **Retention** | Logs 90 days; deliveries 90 days; idempotency keys 24 hours; completed events 7 days |

## 9. Exports (controller: the pharmacy)

A pharmacy's full export (`platform.data_exports`) is a ZIP of every table and file. It is a copy
of everything above for that pharmacy. It is stored in S3 for 7 days behind a 15-minute signed link
and needs the settings-management permission. The export excludes root administrators' notes about
the pharmacy: its status reason and legal hold reason (INT-17).

## Not collected

The platform holds no national ID numbers, dates of birth, card numbers, biometric data (passkeys
store only a public key) or location. The product has no marketing or profiling features and no
automated decisions about people.
