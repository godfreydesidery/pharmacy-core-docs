# Data map

This map shows where personal data lives, how it moves, and which movements cross a border. The
target hosting is AWS **af-south-1 (Cape Town, South Africa)**, described in `pharmacy-core-infra`.
Until the cut-over, production still runs on Railway **US-East** (`iad`). That is a transfer to the
United States, and the move ends it.

```
 Pharmacy (Tanzania)                    AWS af-south-1, South Africa
 ┌──────────────────┐   HTTPS (TLS 1.2+)  ┌───────────────────────────────────────────────┐
 │ browser / till   │────────────────────▶│ ALB + WAF ─▶ web ─▶ internal ALB ─▶ API tasks  │
 │ (offline queue   │◀── presigned S3 ────│                                    │      │    │
 │  in the browser) │    links (15 min)   │   RDS PostgreSQL (Multi-AZ, KMS)  ◀┘      │    │
 └──────────────────┘                     │   S3 files bucket (SSE-KMS)  ◀────────────┘    │
                                          │   CloudWatch Logs (90 days), Secrets Manager   │
                                          │   RDS backups + AWS Backup vault (35 days)     │
                                          └──────────────┬────────────────────────────────┘
                                                         │ HTTPS / SMTP (only the message)
                                    ┌────────────────────▼─────────────────────┐
                                    │ Email / SMS / WhatsApp providers          │
                                    │ (location depends on the provider chosen) │
                                    └───────────────────────────────────────────┘
```

## Where each kind of data is stored

| Store | Location | What | Encryption | Retention |
|---|---|---|---|---|
| RDS PostgreSQL (primary + standby) | af-south-1a/b | Everything in [records-of-processing.md](records-of-processing.md) | At rest: KMS key `pharmacy-<env>-db`. In transit: TLS forced (`rds.force_ssl=1`) | See [retention-schedule.md](retention-schedule.md) |
| RDS automated backups and snapshots | af-south-1 | Full copies of the database | Same KMS key | 35 days (point-in-time recovery) |
| AWS Backup vault | af-south-1 | Daily database snapshots | KMS | 35 days. **No cross-region copy** (off by default) |
| S3 files bucket | af-south-1 | Uploaded files (prescription images, attachments, logos), pharmacy export ZIPs | SSE-KMS, own key; TLS-only policy | Live files while the pharmacy exists; deleted or overwritten versions 90 days; exports 7 days (app), 30 days (backstop) |
| S3 dump bucket | af-south-1 | `pg_dump` archives (the cut-over dump, manual backups) | SSE-KMS; Object Lock (governance) | 35-day lock, expiry after 90 days |
| CloudWatch Logs | af-south-1 | Application logs (masked message addresses; slow SQL statements may contain values) | AWS-managed | 90 days |
| Secrets Manager | af-south-1 | Database passwords, encryption keyring | KMS | Until rotated |
| ECR | af-south-1 | Container images (no personal data) | AES-256 | Last 50 images |
| The browser | The pharmacy's device | Session cookie; queued offline sales until they sync; parked sales | The device's | Until synced, or the browser is cleared |
| Email / SMS / WhatsApp provider | Depends on the provider | The recipient's address and the message | The provider's | The provider's policy ([subprocessors.md](subprocessors.md)) |
| Insurer (payer) | Payer's systems | Claims the **pharmacy** sends | — | Not the platform's processing |

**Not stored:** the platform does not keep ALB access logs or VPC flow logs, both of which would
hold client IP addresses. WAF keeps only sampled requests, for about 3 hours, inside AWS.

## Transfers across borders

| # | From → to | What | Status |
|---|---|---|---|
| T1 | Tanzania → **South Africa** (AWS af-south-1) | All personal and health data | **Planned (cut-over).** Main transfer; needs the steps below |
| T2 | Tanzania → **United States** (Railway US-East) | All personal and health data | **Current.** Ends at the cut-over. Then the Railway database and volume are deleted, with a record |
| T3 | Tanzania/South Africa → the email provider's country | Recipient email and message (receipts, digests, account links) | Depends on the provider chosen (for example US for Resend, Mailtrap or Twilio) |
| T4 | → the SMS/WhatsApp provider's country | Phone number and message | Beem: Tanzania. Africa's Talking: Kenya/regional. Twilio: US (**to confirm per provider**) |
| T5 | South Africa → another AWS region (backup copy) | Database snapshots | **Off.** The infrastructure keeps every backup in af-south-1. Turning it on is a new transfer |

### What the Act requires for T1 (to confirm with counsel, C4)

The secondary sources we read (see [README](README.md)) say:

- **Section 31:** transfer to a country with an adequate legal framework is allowed. **Section 32**
  covers countries without one, under conditions such as consent, the performance of a contract,
  public interest or legal claims. South Africa has a data protection law, the Protection of
  Personal Information Act 2013 (POPIA), with an Information Regulator. Whether the PDPC treats it
  as adequate is for counsel.
- **A permit from the PDPC is needed before the transfer**, applied for on the prescribed form
  (Regulation 20 of the Collection and Processing Regulations, 2023). The application describes the
  applicant, the recipient and the data subjects, the data, the purpose and necessity, the security
  arrangements in the recipient country, the data subjects' consent, and the date of sending. It is
  supported by one of: the recipient country's international data protection commitments, a
  bilateral agreement, or **a contract with the recipient** (here, the AWS agreement and its data
  processing addendum).
- **Conditions after the permit:** only to the named recipient, only for the stated purpose, no
  onward transfer without the PDPC's approval, and processing abroad must not break Tanzanian law.

### Safeguards to document for the permit

1. **Contract:** the AWS Customer Agreement and AWS Data Processing Addendum. Keep a dated copy of
   the versions in force and note the AWS contracting entity.
2. **Region lock:** every store is in af-south-1. Cross-region backup copy is off.
   **Proposal:** add an AWS Organizations service control policy that denies other regions, which
   makes the region lock a technical control as well.
3. **Encryption:** KMS keys owned by the platform's AWS account for the database, backups, files
   and secrets; TLS 1.2+ on every connection; the database refuses plain connections.
4. **Access:** private subnets, no public database. Each task gets least-privilege roles, and the
   API never holds the owner password. Deploys come only from CI on `main` (OIDC). Administrators
   reach the database only through Session Manager. Every action in a pharmacy is in the audit
   trail.
5. **Isolation:** forced row-level security between pharmacies, enforced by the database.
6. **Retention and deletion:** [retention-schedule.md](retention-schedule.md), whole-pharmacy
   export and erasure.
7. **Breach handling:** [breach-response.md](breach-response.md).
8. **Sub-processors:** [subprocessors.md](subprocessors.md); onward transfers need PDPC approval.

## Registration with the PDPC (to confirm with counsel, C2)

The sources say **every controller and every processor must register** with the PDPC, and that a
registration lasts five years. For Pharmacy Core this likely means:

- **The platform** registers as a **processor** (for pharmacies' data) and as a **controller** (for
  its own: pharmacy account contacts, staff identities it controls, security logs).
- **Each pharmacy** registers as a **controller**. The platform can help with a short guide and with
  the description of processing from [records-of-processing.md](records-of-processing.md). The
  processor agreement asks each pharmacy to confirm its registration.
- The transfer permit (above) is separate from registration.

Track the registration numbers, dates and renewal dates in the platform's compliance register
(owner and DPO).
