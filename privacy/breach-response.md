# Personal data breach response

A **personal data breach** is any security incident that leads to personal data being lost,
destroyed, changed, disclosed or accessed without authority. It does not matter whether the cause
is an attack, a mistake or a failure. Examples: one pharmacy seeing another's data, a leaked export
link or database dump, a stolen administrator laptop with a dump on it, a misconfigured bucket,
ransomware, an email sent to the wrong person, a sub-processor's breach.

## What the law says (to confirm with counsel, C5)

- The Act requires the **controller** to notify the **PDPC** of a breach **"without undue
  delay"** (section 27(5) as cited by the sources we read). The sources say the Act sets **no fixed
  number of hours** and that notice is **not conditional on the level of harm**: any breach
  affecting personal data is notifiable.
- The PDPC has a **prescribed reporting format**. Per the sources, it covers the controller or
  processor's details, a summary and an assessment of the breach, the notifications made, the root
  cause, the technical and organisational measures, areas for improvement, supporting documents,
  and a signed declaration.
- A **processor must inform the controller promptly**, so the controller can notify in time.
- The Act imposes **no clear duty to tell data subjects**. We tell them anyway when they are
  likely to be harmed (good practice; **counsel to confirm**).

**Our targets.** "Without undue delay" is not a number, so we set strict ones:

| Step | Target (from becoming aware) |
|---|---|
| Platform (as processor) tells the affected pharmacies | **24 hours** |
| Notice to the PDPC: by the pharmacy (controller), drafted by the platform; by the platform for its own data | **72 hours** at the latest, even if the facts are incomplete (send what is known, then follow up) |
| Affected individuals, where harm is likely | As soon as the facts allow, normally within **72 hours** of confirmation |

72 hours matches regional practice and the GDPR. It is our choice, not a figure from the Act:
**check with counsel**.

"Becoming aware" means the moment someone reasonably believes a breach happened, not the moment
the investigation ends.

## Roles

| Role | Who | Does |
|---|---|---|
| Incident lead | The on-call engineer who sees it, until handed over | Contains, keeps evidence, runs the timeline |
| DPO | See [dpo-role.md](dpo-role.md) | Decides whether it is a breach, drafts the notices, keeps the breach register |
| Owner / management | The company's director | Approves notices; speaks for the company |
| Counsel | External Tanzanian lawyer | Confirms the notices and any duty to individuals |
| Pharmacy contact | Each affected pharmacy's administrator | Receives our notice; as controller, files with the PDPC (we help) |

## The steps

### 1. Detect and record (hour 0)

Sources: CloudWatch alarms and logs, the audit trail (`audit.entries`), the WAF, AWS GuardDuty and
CloudTrail (when enabled), a pharmacy's report, a provider's notice, a researcher's report.

Open an incident record **immediately**: time of awareness, who reported it, what is known.
Write every later action into it with its time.

### 2. Contain (hours 0-4)

Pick the measures that fit:

- Revoke sessions: the affected users' sessions, or every session of a pharmacy, from the
  database.
- Suspend the pharmacy or the account (suspension kinds), or put the pharmacy under a **legal
  hold** so nothing is purged or erased while evidence is needed.
- Rotate exposed secrets: database passwords (`pharmacy-core-infra/docs/secrets.md`), provider
  credentials, the deploy roles. Do **not** rotate the encryption key until the keyring supports
  it (OPS-5).
- Close the hole: revert the deploy (CI redeploys the previous SHA), block an IP in the WAF, make
  a bucket private, disable a provider.
- **Keep evidence:** do not delete logs or tasks. Copy the relevant CloudWatch logs and audit rows
  to the incident record. Take an RDS snapshot if data may have been changed.

### 3. Assess (hours 4-24)

Answer these, and write the answers down:

- What data was involved? Health data (prescriptions, diagnosis codes, controlled-drug sales),
  phone numbers, sign-in data?
- How many people, and which pharmacies? Use the audit trail, application logs and S3 access to
  bound it.
- Was the data encrypted and the key safe? Encrypted data with a safe key lowers the risk, but the
  breach must still be notified.
- What harm could follow: discrimination or stigma (health), fraud, account takeover?
- Is it still going on?

### 4. Notify (by hour 24 for pharmacies, hour 72 for the PDPC)

- **Pharmacies (within 24 h):** what happened, which of their data, what we did, what they should
  do, and our draft of the PDPC notice for them.
- **PDPC (within 72 h):** on the prescribed form, filed by each affected pharmacy as controller and
  by the platform for its own data. Counsel should confirm whether the platform may file on the
  pharmacies' behalf. Send what is known; follow up with the rest.
- **Individuals (where harm is likely):** plain language, Swahili and English: what happened, what
  it means for them, what we are doing, who to contact.
- **Sub-processor's breach:** they must tell us at once (per their DPA); we then follow the same
  steps.

### 5. Recover and learn (within 2 weeks)

- Fix the root cause. Add a test or an alarm so it is caught next time.
- Write the post-incident report: timeline, cause, impact, notices sent, improvements. It becomes
  the "root cause" and "areas for improvement" sections of the PDPC follow-up.
- Update the **breach register**. Every incident goes in, including ones judged not to be
  breaches, with the reason.

## Breach register (template)

| Field | |
|---|---|
| Incident id, date and time of awareness | |
| Reported by / detected by | |
| Description | |
| Data and people affected (categories, approximate numbers, pharmacies) | |
| Containment actions and times | |
| Breach? (yes/no, why) | |
| Pharmacies notified (when) | |
| PDPC notified (when, by whom, reference) | |
| Individuals notified (when, how) | |
| Root cause | |
| Improvements and owner | |
| Closed (date, DPO) | |

## Practise

Run a tabletop exercise **twice a year**, for example "an export link was posted in a WhatsApp
group" or "a cashier saw another pharmacy's customers". Time each step against the targets above.
