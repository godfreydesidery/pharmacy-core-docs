# Data subject requests

People whose data a pharmacy holds have rights under the Act. The sources we read list: being
informed, access, rectification (correction), erasure, restriction, objection (including to
direct marketing and profiling), and not being subject to solely automated decisions.

**Who answers.** For customers, patients, prescribers and the pharmacy's staff, the **pharmacy is
the controller** and answers the request. The platform, as processor, helps under the processor
agreement. For the platform's own data (pharmacy account contacts, root administrators, personal
staff accounts as decided under C1), the platform answers.

**Deadline.** We found no fixed response time in the sources. Until counsel confirms (**C7**),
answer within **30 days** of a verified request, and acknowledge within **5 working days**. No fee.

## What the product supports today

| Right | One whole pharmacy | One person (customer, patient, staff member) |
|---|---|---|
| Access (a copy) | **Built.** Full export: `POST /api/v1/data-export` (settings-management permission). A background ZIP of every table and file, a signed link for 15 minutes, kept 7 days | **Manual.** Staff view and print the customer's record and sales; the customer list can be exported with `data.export`. **Built (Batch 9):** `POST /customers/{uid}/data-export`, a ZIP of everything about the person, audited and logged as an ACCESS request |
| Correction | n/a | **Partly built.** Customers, suppliers, staff profiles and memberships can be edited. Completed sales, prescriptions on sales, claims and ledger entries are **append-only**: correct them with a note or a return, never by editing |
| Erasure | **Built.** Root administrator: close the pharmacy (`POST /api/v1/platform/tenants/{uid}/close`), 30-day cooling-off (`pharmacy.tenancy.erase-after-days`), then erase (`POST /api/v1/platform/tenants/{uid}/erase`). Every row and file of the pharmacy goes; the tenant row stays as a tombstone without contact details; the platform's billing records stay. A **legal hold** blocks it | **Built (Batch 9):** `POST /customers/{uid}/anonymise` removes name, phone, email, address and insurance number (also from their sales, claims and messages) and keeps the transaction records, because sales and ledgers are append-only and needed by law (**Counsel C10**). Refused while the account has a balance or a legal hold applies |
| Restriction | Suspension (the pharmacy cannot work in it) or a legal hold | **Manual.** Deactivate the customer. Not a full restriction flag |
| Objection | n/a | The product does no marketing, no profiling and no automated decisions. SMS receipts are a pharmacy setting (`sms_receipts`), and notification preferences exist. **Built (Batch 9):** consent records per customer; SMS receipts stop when the customer withdraws |
| Automated decisions | n/a | None in the product |

**Backups and versions.** Erased data also stays in database backups for up to 35 days, and as
noncurrent S3 file versions for up to 90 days. Say so in the answer to an erasure request. Backups
are never restored to bring erased data back. If a backup must be restored for another reason,
repeat the erasures done since that backup.

## Procedure

1. **Receive.** A request can come in any form, to the pharmacy or to the platform
   (`dpo@...`). Record it: date, requester, right asked for, pharmacy.
2. **Route.** If it reached the platform but concerns a pharmacy's customer, forward it to that
   pharmacy's administrator **within 2 working days** and tell the requester. The pharmacy decides,
   and we help.
3. **Verify identity.** Before giving data or changing anything, confirm the requester is the
   person, or their parent or guardian for a child (**C13**), or a representative with written
   authority. Match the phone number or email on the record, or check ID at the pharmacy counter.
   Never send health data to an unverified address.
4. **Check limits.** A request may be refused, fully or partly, only on grounds the Act allows:
   legal retention duties (**C8**), legal claims, other people's rights. Record the reason.
5. **Act.**
   - *Access:* collect the person's records: the customer record, sales with patient and
     prescription fields, insurance membership and claims, messages sent to them, files linked to
     them. Until Batch 9 an engineer pulls them with a reviewed, read-only query scoped to the
     pharmacy (`set pharmacy.tenant_uid`). The pharmacy reviews the copy before it goes out, to
     remove other people's data.
   - *Correction:* edit in the app; for append-only records, add a correcting note or document.
   - *Erasure:* until Batch 9, remove contact and identifying fields from the customer record and
     deactivate it, and record what remains and why (legal retention). After Batch 9, use the
     anonymise action.
   - *Objection / restriction:* deactivate the customer, turn off messages to them, and note it.
6. **Answer** in plain Swahili or English, within the deadline. Say what was done, what was not,
   why, and how to complain to the PDPC.
7. **Log** the request and its outcome in the request register (date received, verified, answered,
   outcome). The DPO reviews the register quarterly.

## Request register (template)

| Received | Requester (no health data here) | Pharmacy | Right | Verified (how) | Answered | Outcome / reason |
|---|---|---|---|---|---|---|
| | | | | | | |
