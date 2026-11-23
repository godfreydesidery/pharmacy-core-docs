# Data Protection Officer (DPO)

The sources we read say Tanzania's Act requires controllers and processors to appoint a **Data
Protection Officer**. They also say the DPO oversees security controls and compliance, handles
data subject requests and complaints, and reports to the PDPC, possibly with **quarterly
compliance reports**. **Counsel (C6):** confirm who must appoint one, any required qualifications,
whether the DPO must be notified to or registered with the PDPC, and the reporting duties.

This page describes the platform's DPO. Each pharmacy, as a controller, may need its own DPO. For a
small pharmacy that may be the owner or the superintendent pharmacist. The platform's DPO helps
them, but does not replace them.

## Position

- Appointed by, and reports to, the company's management (the owner/director).
- Can be an employee or an external service, but **must not** hold a role that decides on the same
  processing they oversee: not the lead engineer who runs production, and not the head of sales.
  If the company is too small for that, record the conflict and how it is handled.
- Has the time, budget and access to do the job. Is not penalised for advice the company does not
  like.
- Is contactable: a published address, for example `dpo@<company domain>`, reaches the DPO
  directly. It appears in the privacy notice, the processor agreement and the PDPC registration.

## Responsibilities

1. **Registration and permits.** Keep the platform's PDPC registration current (5-year validity;
   renew in good time). Prepare and hold the cross-border transfer permit for South Africa. Help
   pharmacies with theirs ([data-map.md](data-map.md)).
2. **Records.** Keep [records-of-processing.md](records-of-processing.md),
   [data-map.md](data-map.md), [subprocessors.md](subprocessors.md) and
   [retention-schedule.md](retention-schedule.md) true to the product. Review them at each release
   that adds personal data. A new column holding personal data is a change the DPO signs off.
3. **Requests and complaints.** Run [data-subject-requests.md](data-subject-requests.md). Answer
   pharmacies' privacy questions. Be the contact for the PDPC.
4. **Breaches.** Lead the assessment and notices in [breach-response.md](breach-response.md). Keep
   the breach register.
5. **Assessments.** Carry out a privacy impact assessment before a high-risk change: new health
   data, a new country, a new sub-processor, the per-person rights work (Batch 9), consent records.
6. **Sub-processors.** Check each new provider's location, DPA and transfer status before it is
   used. Give pharmacies 30 days' notice of changes.
7. **Training.** Brief everyone with production access, at least once a year and at joining.
   Cover what is personal and health data, never copying production data to laptops or chat, and
   reporting incidents at once.
8. **Reports.** Report to management at least quarterly (requests handled, incidents, open risks,
   registration dates), and to the PDPC as the regulations require (**counsel C6**).
9. **Audits.** Once a year, review access: who has AWS administrator access, root administrators,
   the deploy roles, and that the region lock holds.

## What the DPO needs from engineering

- A read of the audit trail and logs during an incident (through an engineer; the DPO does not
  need production credentials).
- Notice of changes that add personal data, a new provider or a new region before they ship.
- The yearly access list.

## Appointment record

| | |
|---|---|
| Name | **TBD (owner to appoint)** |
| Contact | `dpo@...` **TBD** |
| Appointed on | |
| Notified to the PDPC on | (if required, C6) |
| Deputy | |
