# Privacy pack

Groundwork for Tanzania's **Personal Data Protection Act, 2022** (Act No. 11 of 2022, "the Act" or
"PDPA") and its 2023 regulations. This pack covers OPS-15 and STD-6 of the adversarial review of
23 Nov 2026. It describes Pharmacy Core as it is built today, from the code in `pharmacy-core-api`
and the AWS design in `pharmacy-core-infra`. It is **not legal advice**. Every point marked
**Counsel** must be confirmed by a Tanzanian data protection lawyer before the documents are used
with pharmacies or the Personal Data Protection Commission (PDPC).

| Document | What it answers |
|---|---|
| [records-of-processing.md](records-of-processing.md) | What personal and health data we hold, about whom, why, on what lawful basis, for how long |
| [data-map.md](data-map.md) | Where the data lives and moves (AWS af-south-1, providers), cross-border transfers, PDPC registration and permits |
| [subprocessors.md](subprocessors.md) | Who processes data for us (AWS, message providers), with placeholders where not chosen |
| [breach-response.md](breach-response.md) | What to do, and by when, when personal data may have leaked or been lost |
| [dpo-role.md](dpo-role.md) | The Data Protection Officer's role |
| [data-subject-requests.md](data-subject-requests.md) | Access, correction, erasure and objection requests: what the product supports today and what is manual |
| [retention-schedule.md](retention-schedule.md) | How long each kind of data is kept, aligned with the code's retention settings |
| [processor-agreement-outline.md](processor-agreement-outline.md) | Outline of the data processing agreement between the platform and each pharmacy |

## The roles in one paragraph

Each **pharmacy** decides why and how its customers', patients', prescribers' and staff's data is
used. It is the **controller** of that data. **The platform** (the company that runs Pharmacy Core)
stores and processes it on the pharmacy's behalf, and is its **processor**. The platform is a
controller of its own data: the pharmacies' account and billing contacts, root administrators,
and the security logs it keeps to run the service. **Counsel (C1):** confirm this split,
especially for staff with a personal account used across several pharmacies.

## Status (23 Nov 2026)

| Item | State |
|---|---|
| Hosting | Moving from Railway US-East to AWS **af-south-1 (Cape Town, South Africa)** (owner's decision). The move is a cross-border transfer out of Tanzania: see [data-map.md](data-map.md) |
| Whole-pharmacy export | **Built.** Background ZIP of every table and file, signed link for 15 minutes, kept 7 days |
| Whole-pharmacy closure and erasure | **Built.** Close, 30-day cooling-off, erasure by a root administrator, legal hold stops it |
| Nightly retention | **Built.** Sessions, codes, invitations, message logs, deliveries, events, exports, audit months |
| Per-person rights (access, correction, erasure, objection for one customer or patient) | **Planned, Batch 9 (STD-5).** Manual procedure until then |
| Consent records | **Planned, Batch 9 (STD-5)** |
| PDPC registration, DPO, transfer permit | **Not done.** Owner and counsel |

## Points for counsel

| # | Question |
|---|---|
| C1 | Controller/processor split above, including staff personal accounts that span pharmacies and the platform's root administrators who can see pharmacy data for support |
| C2 | Registration with the PDPC: does the platform register as a processor (and a controller for its own data), and must every pharmacy register as a controller? Fees, the 5-year validity, renewal, and whether we may help pharmacies register |
| C3 | Lawful basis for health data (prescriptions, patient names, controlled-drug sales, diagnosis codes on insurance claims). Is it sensitive personal data needing written consent, or covered by an exception (medical care under a health professional, a legal obligation under pharmacy and medicines law)? |
| C4 | The transfer to South Africa: section 31 (adequate country) or 32, whether South Africa's POPIA counts as adequate, the transfer permit (Regulation 20) and who applies (each pharmacy, the platform, or both). Whether data subjects' consent is needed, since the permit form asks for it. Must this be in place **before** the cut-over? |
| C5 | Breach notice: the Act says "without undue delay" (s. 27(5)) with no fixed hours. Is our 72-hour target right, what is the PDPC's prescribed form, and must data subjects be told? |
| C6 | DPO: who must appoint one, whether the DPO must be registered with or reported to the PDPC, required qualifications, and any periodic (for example quarterly) compliance reports |
| C7 | Deadlines and fees for data subject requests, and the identity checks expected |
| C8 | Legal minimum retention under sector law: prescriptions and controlled-drug registers (TMDA, Pharmacy Council, drug control law), tax records (Tax Administration Act), insurance claims, employment records |
| C9 | The 7-year default (2-year floor) on the audit trail: justified for fraud and dispute evidence? |
| C10 | Erasure of one person when sales and the ledgers are append-only: is anonymisation (name removed, record kept) enough? |
| C11 | Message providers outside Tanzania (for example email in the US): each is a further transfer |
| C12 | Section and regulation numbering: the Act was revised as Cap. 44 R.E. 2023, and sources disagree on the regulations' Government Notice numbers (GN 449B/449C of 2023 versus GN 349/350 of 2023) |
| C13 | Patients who are children: consent by a parent or guardian, and how a pharmacy records it |
| C14 | The processor agreement's terms: liability, audit, governing law, sub-processor approval |

## Sources read (Nov 2026)

These are secondary sources, read to frame the questions. They are not a substitute for counsel or
for the official Gazette text.

- DLA Piper, *Data Protection Laws of the World: Tanzania*:
  <https://www.dlapiperdataprotection.com/index.html?t=law&c=TZ>
- Clyde & Co, *Cross-border personal data transfers* (Oct 2024):
  <https://www.clydeco.com/en/insights/2024/10/cross-border-personal-data-transfers>
- Clyde & Co, *Notification obligations arising from personal data breaches in Tanzania* (Feb
  2026): <https://www.clydeco.com/en/insights/2026/02/notification-obligations-arising-from-personal-dat>
- Bowmans, *Tanzania: regulations made under the Personal Data Protection Act 2022*:
  <https://bowmanslaw.com/insights/data-protection/tanzania-regulations-made-under-the-personal-data-protection-act-2022/>
