# 0009 Integration ports and CloudEvents webhooks

- Status: accepted
- Date: 2026-11-02

## Context
The system has to integrate with payment providers (M-Pesa, Airtel Money, card terminals, insurers), tax authorities (TRA VFD, KRA eTIMS), external POS terminals, ERPs and e-commerce. Which providers depends on the market and the tenant.

## Decision
- **Outbound ports.** Provider calls are outbound ports of the module that needs them, e.g. `sales.ports.outbound.PaymentAuthorizations` and `FiscalReceipts`.
- **Adapters.** Each port is implemented in composition. A property selects the adapter: `pharmacy.integrations.payments.provider`, `pharmacy.integrations.fiscal.provider`. The defaults are `manual` and `none`.
- **Outbound events.** The `integrations` module sends domain events to registered webhooks. They are CloudEvents 1.0 JSON, signed with `X-Pharmacy-Signature: sha256=<HMAC-SHA256>`, retried, and every delivery is logged.
- **Inbound.** External systems use the versioned REST API `/api/v1`, with `Idempotency-Key` for safe retries of sales.

## Consequences
- Adding a provider means adding one adapter class and setting a property. No module changes.
- Webhooks are at least once. Receivers should deduplicate by CloudEvent `id`.
