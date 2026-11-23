# Sub-processors

These are the companies that process pharmacies' personal data on the platform's behalf. The
processor agreement ([processor-agreement-outline.md](processor-agreement-outline.md)) lets each
pharmacy see this list. The platform gives at least **30 days' notice** before adding or replacing
a sub-processor, and a pharmacy may object. Placeholders (**TBD**) are providers not chosen yet.
The code already supports them; the platform administrator picks one in the app's Message
providers pages.

| Sub-processor | Service | Data | Where processed | Safeguards | Status |
|---|---|---|---|---|---|
| **Amazon Web Services** (contracting entity per the AWS Customer Agreement: **to confirm**) | Hosting: ECS Fargate, RDS PostgreSQL, S3, KMS, Secrets Manager, CloudWatch, Elastic Load Balancing, WAF, ECR, AWS Backup, Certificate Manager | All personal and health data | **af-south-1, Cape Town, South Africa** | AWS Customer Agreement + Data Processing Addendum; encryption with the platform's KMS keys; region lock | **Chosen** (owner, Nov 2026). Cut-over pending |
| Railway Corporation | Current hosting (database, API, files) | All | **US-East** | Railway terms | **Leaving** at the cut-over; then delete its database and volume, and record the deletion |
| **Email provider: TBD** (Resend and Mailtrap were used in testing) | Sends account emails (reset links, invitations, codes), digests, receipts | Recipient email, subject and body | TBD (US for those two) | Provider DPA; TLS (SMTP submission); no message text kept by Pharmacy Core after an account message is sent | Not chosen for production |
| **SMS provider: TBD**, one of Beem (Tanzania), Africa's Talking, Twilio | SMS receipts (if the pharmacy turns them on), sign-in codes | Phone number, message | Beem: Tanzania; Africa's Talking: Kenya/regional; Twilio: US (**to confirm**) | Provider DPA | Not chosen |
| **WhatsApp provider: TBD** (Twilio) | WhatsApp messages | Phone number, message | US (Twilio, and Meta for WhatsApp) | Provider DPA | Not chosen |
| **Payments provider: TBD** (mobile money) | Push payment requests | Phone number, amount, reference | TBD | Provider DPA | `manual` today, no provider |
| **Error tracker: TBD** (OPS-7, planned) | Crash reports | Must be configured to send no personal data | TBD | Scrubbing before sending | Not chosen |

**Not sub-processors:**

- **GitHub** (source code and CI) holds no personal data. Production data and secrets never go
  into repositories or CI logs.
- **Insurers** receive claims from the pharmacy, as the pharmacy's own disclosure.
- **The Tanzania Revenue Authority** (fiscal receipts, planned) is a public authority that receives
  data by law.

**Before choosing a provider:** check where it processes and stores data, its DPA, its breach
notice terms, and whether using it is a transfer that needs PDPC approval (**Counsel C11**). Prefer
a provider in Tanzania or South Africa when the service is equal.
