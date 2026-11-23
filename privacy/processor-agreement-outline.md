# Data processing agreement: outline

This is an outline of the agreement between **the platform** (the company that runs Pharmacy
Core, the **processor**) and **each pharmacy** (the **controller**). It lists what the agreement
must cover, so counsel can draft it (**C14**). It would sit alongside the subscription terms and
be accepted when a pharmacy signs up. Existing pharmacies would accept it before the cut-over to
AWS.

## 1. Parties and roles

- The pharmacy: legal name, TIN, PDPC registration number (or a commitment to register; **C2**),
  and its contact for privacy matters (its DPO if it has one).
- The platform: legal name, PDPC registration number, DPO contact.
- Roles: the pharmacy is the controller for the data of its customers, patients, prescribers,
  suppliers and staff. The platform is its processor. The platform is a controller only for its
  own account, billing and security data (**C1**).

## 2. Subject matter, duration, nature and purpose

- **Subject matter:** providing Pharmacy Core (POS, dispensing, stock, purchasing, insurance
  claims, accounting, reporting, notifications).
- **Duration:** the subscription, plus the closure, cooling-off (30 days) and erasure period, plus
  backup expiry (up to 35 days for backups, up to 90 days for file versions).
- **Purpose:** only to provide the service as the pharmacy instructs: its use of the product, its
  settings, and written instructions.
- **Data and data subjects:** a summary of [records-of-processing.md](records-of-processing.md),
  including **sensitive health data** (prescriptions, patient names, diagnosis codes,
  controlled-drug sales).

## 3. The platform's obligations

1. Process only on the pharmacy's documented instructions, and tell the pharmacy if an
   instruction seems unlawful.
2. **Confidentiality:** staff with access are bound to confidentiality and trained. Production
   access is limited and logged.
3. **Security measures** (an annex listing them): hosting in AWS af-south-1; encryption at rest
   (KMS) and in transit (TLS 1.2+, database TLS forced); forced row-level isolation between
   pharmacies; least-privilege roles; no public database; deploys from CI only; audit trail; 2FA
   required for root administrators; nightly retention; backups for 35 days; incident response.
4. **Sub-processors:** the current list ([subprocessors.md](subprocessors.md)) is approved on
   signature. The platform gives **30 days' notice** of a new or replaced sub-processor, and the
   pharmacy may object. If the objection cannot be resolved, the pharmacy may terminate without
   penalty. The platform imposes the same data protection obligations on each sub-processor and
   stays responsible for them.
5. **Cross-border transfers:** data is stored in South Africa (af-south-1). The platform obtains
   and keeps the PDPC transfer permit or other required approval, and the pharmacy co-operates in
   any application that must be its own (**C4**). No transfer to another country or onward
   recipient without the pharmacy's notice and the PDPC's approval where required.
6. **Assistance with data subject requests:** the platform provides the tools (export; per-person
   tools from Batch 9) and, until then, the manual help in
   [data-subject-requests.md](data-subject-requests.md), within a set time (for example 10 working
   days).
7. **Breaches:** the platform tells the pharmacy **without undue delay, and within 24 hours** of
   becoming aware of a breach affecting its data. The notice says what is known (nature,
   categories, approximate numbers, likely consequences, measures taken) and is followed up. The
   platform drafts the PDPC notice for the pharmacy ([breach-response.md](breach-response.md),
   **C5**).
8. **Assistance with impact assessments and PDPC enquiries.**
9. **End of service:** the pharmacy can export all its data at any time, including while closed.
   After closure and the 30-day cooling-off, the platform erases it (legal hold excepted) and
   confirms in writing. Backups and file versions expire within 90 days. The platform keeps only
   the tombstone and its own billing records.
10. **Records and audit:** the platform keeps records of its processing and makes available the
    information needed to show compliance. Audits are by questionnaire or report first, and on site
    only with reasonable notice, at the pharmacy's cost, and without access to other pharmacies'
    data.

## 4. The pharmacy's obligations

- Have a lawful basis for the data it enters, including health data (consent or another ground;
  **C3**). Inform its customers and patients (a privacy notice at the counter and on receipts;
  **template to be provided**).
- Register with the PDPC as a controller (**C2**), and appoint a DPO if required (**C6**).
- Control its own users: give permissions with care, remove people who leave, and protect
  sign-ins (2FA available).
- Enter only the data it needs. For example, no diagnoses in free-text notes unless required.
- Answer its data subjects' requests, with the platform's help.
- Tell the platform at once of any suspected breach on its side (a shared password, a lost
  device).

## 5. Legal holds and authorities

- At the request of the pharmacy, a court or a regulator, the platform can place the pharmacy
  under a **legal hold**: nothing is purged or erased until the hold is lifted.
- If an authority demands the pharmacy's data, the platform tells the pharmacy first unless the law
  forbids it, and discloses only what is legally required.

## 6. Liability, law and term (counsel to draft, C14)

- Liability and indemnity between the parties, and their link to the subscription terms' caps.
- Governing law (Tanzania) and dispute resolution.
- Precedence: this agreement prevails over the subscription terms on data protection.
- Term: as long as the platform holds the pharmacy's data.

## Annexes

1. Description of processing (from [records-of-processing.md](records-of-processing.md)).
2. Security measures.
3. Sub-processors ([subprocessors.md](subprocessors.md)).
4. Retention ([retention-schedule.md](retention-schedule.md)).
5. Contacts: both DPOs, the breach notice address.
