# Retention schedule

How long each kind of data is kept, and what deletes it. Periods marked **code** are the defaults
of `Retention` (`platform/.../persistence/retention/Retention.java`, properties
`pharmacy.retention.*`). They are enforced by the database functions `platform.purge_expired` and
`audit.drop_partitions_before`, run **nightly at 01:40 UTC on one instance**, and covered by
`RetentionTest`.

Three rules apply to the whole nightly purge:

- Every period is counted back from the database's `now()` at the latest, so a caller cannot pass
  a future date to purge early.
- A pharmacy under a **legal hold** keeps everything until the hold is lifted. An audit month is
  kept whole while any held pharmacy has entries in it.
- The audit trail is never cut below **2 years** (`audit.minimum_retention()`), whatever is
  configured.

## Account and sign-in data

| Data | Table | Kept | Deleted by | Setting |
|---|---|---|---|---|
| Sign-in sessions (with IP and user agent) | `access.sessions`, `access.session_pharmacies` | **30 days** after expiry or revocation | nightly purge (code) | `sessions-days` 30 |
| Password reset links, phone codes, email changes, sign-in challenges, passkey ceremonies | `access.password_resets`, `phone_codes`, `email_changes`, `sign_in_challenges`, `webauthn_ceremonies` | **7 days** after expiry | nightly purge (code) | `codes-days` 7 |
| Invitations | `access.invitations` | **90 days** after expiry, acceptance or revocation | nightly purge (code) | `invitations-days` 90 |
| User accounts, memberships | `access.users`, `access.memberships` | While active. A former member's `former_*` snapshot stays with the former employer | Pharmacy erasure (its staff logins); no time limit yet | **Gap:** decide a period for left staff (**Counsel C8**) |

## Messages and integrations

| Data | Table | Kept | Deleted by | Setting |
|---|---|---|---|---|
| Sent or failed messages (email, SMS, WhatsApp) | `notifications.messages` | **180 days** after creation. Account messages (reset, invitation, codes) keep **no text** once sent | nightly purge (code) | `messages-days` 180; `keep-account-messages: false` |
| In-app notices | `notifications.inbox` | **180 days** after being read | nightly purge (code) | `inbox-days` 180 |
| Webhook deliveries (payload text) | `integrations.deliveries` | **90 days** (delivered or failed) | nightly purge (code) | `deliveries-days` 90 |
| Idempotency keys and stored responses | `platform.idempotency_keys` | **24 hours** | nightly purge (code) | `pharmacy.idempotency.lifetime` |
| Completed internal events | `platform.event_publication` | **7 days** after completion | nightly purge (code) | `events-days` 7 |

## Exports and files

| Data | Where | Kept | Deleted by | Setting |
|---|---|---|---|---|
| Export ZIP | S3 files bucket | **7 days**, then the file is deleted | nightly retention task (`DataLifecycleRetention`) | `pharmacy.exports.keep` P7D |
| Export job record | `platform.data_exports` | **30 days** after it ended | nightly purge (code) | `exports-days` 30 |
| Export ZIP (backstop) | S3 lifecycle | 30 days | S3 lifecycle (infra) | Matches only once the API stores exports under `exports/` or tags them; see the infra README |
| Uploaded files (prescriptions, attachments, logos) | S3 files bucket | While the pharmacy exists | Pharmacy erasure; failed deletes retried nightly | — |
| Overwritten or deleted file versions | S3 (versioning) | **90 days** | S3 lifecycle (infra) | `noncurrent_version_days` 90 |

## Business and health records

| Data | Kept | Deleted by |
|---|---|---|
| Sales (incl. patient, prescriber and prescription fields), returns, shifts, vouchers | While the pharmacy uses the platform | Pharmacy erasure. Per-person anonymisation is planned (Batch 9) |
| Customers, insurance members and claims, receivables | Same | Same |
| Stock, purchasing, finance and ledger entries | Same | Same |

**Sector law may require a minimum** for prescriptions, controlled-drug registers, tax and
accounting records (often quoted as 5 years for tax records) and insurance claims (**Counsel C8**).
The platform does not delete these on a timer. The pharmacy, as controller, decides, and erasure
is its decision at closure. A per-pharmacy retention policy for business records is a possible
later feature.

## Audit trail

| Data | Kept | Deleted by | Setting |
|---|---|---|---|
| `audit.entries` (who did what, with before/after values), monthly partitions | **7 years** by default, never less than **2 years** | Nightly: whole months older than the period are dropped, except months holding entries of a pharmacy under legal hold | `audit-years` 7; floor `audit.minimum_retention()` 2 years (**Counsel C9**) |

## Pharmacy closure and erasure

| Step | When |
|---|---|
| Closed (no one works in it; admins can still read and export; can be reopened) | On the owner's request or at the end of a lapsed account |
| Cooling-off | **30 days** (`pharmacy.tenancy.erase-after-days`) |
| Erasure (root administrator) | After the cooling-off, unless under legal hold: every row and file of the pharmacy, its internal events, its staff logins |
| What stays | The tenant row as a **tombstone** without contact details; the platform's billing records (subscriptions, invoices) as tax records (**Counsel C8**) |

## Infrastructure

| Data | Kept | Where set |
|---|---|---|
| Application logs (CloudWatch) | **90 days** | `log_retention_days` (infra) |
| Database logs (slow statements, DDL) | **90 days** | database module (infra) |
| Database point-in-time backups | **35 days** | `db_backup_retention_days` (infra) |
| AWS Backup daily snapshots | **35 days**, in af-south-1 only | database module (infra) |
| `pg_dump` archives in the dump bucket | Locked 35 days, expire after **90 days** | storage module (infra) |
| Railway (old hosting) | Stopped at the cut-over; deleted after **30 days**, with a deletion record | cut-over plan |

**Erasure and backups.** After a pharmacy or a person is erased, copies survive in backups for up
to 35 days and in noncurrent S3 versions for up to 90 days. Then they expire. Backups are kept for
recovery only and never searched or restored to bring erased data back. This should be stated in
the processor agreement and in answers to erasure requests.
