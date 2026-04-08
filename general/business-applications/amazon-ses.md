# Amazon SES

## Service Overview
Amazon SES (Simple Email Service) is a scalable cloud email platform for sending transactional and bulk email with reputation and deliverability features.

## Tasks this service can solve
- Transactional email (password resets, notifications)
- Bulk marketing campaigns and newsletters
- Inbound email processing and routing

## Alternatives
| Name | Type | Short description | Pro vs SES | Cons vs SES | Price comparison |
|---|---|---:|---|---|---|
| SendGrid / Mailgun | Commercial | Email delivery providers | Rich campaign tools & dashboards | External dependency | Often higher per-email fees for advanced features |
| Self-hosted Postfix / Mail servers | OSS | Full control of mail stack | Full control over deliverability | Complex ops and reputation management | Ops cost vs SES managed pricing |

## Limitations
- Deliverability requires management of sending practices, DKIM/SPF, and warm-up.
- Regional availability of certain sending features (e.g., dedicated IPs) varies.

## Price info
- Charged per-email sent plus data transfer; inbound email receiving and mailbox simulation may have separate charges.

## Network & Multi-region considerations
- SES is regional; manage sending regions for compliance or use cross-region setups for redundancy. Integrates with SNS, Lambda, and S3.

## Popular use cases
- Transactional email for web apps
- Newsletter and marketing sends (with reputation management)
- Inbound processing for automated workflows

## When Not to Use This Service
- Not ideal if you require advanced campaign management and marketing automation features out-of-the-box; consider SendGrid/Mailgun or marketing platforms for those capabilities.

## DR strategy
- Maintain sending configuration, templates, and suppression lists in version control and replicate sending domains and identities across regions/accounts where supported. Use multi-region backups of inbound email storage (S3) and monitor deliverability metrics actively.
