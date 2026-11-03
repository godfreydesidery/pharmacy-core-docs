# 0006 Transactional outbox and after-commit reactions

- Status: accepted
- Date: 2026-11-02

## Context
A sale must post to the ledger, update the dashboard and notify webhooks. Doing all of that in the sale's transaction would couple the modules and slow down the POS.

## Decision
- **Outbox.** Modules publish events with `DomainEventPublisher` inside their transaction. Spring Modulith's JDBC event publication registry (`platform.event_publication`) stores them with the transaction.
- **Reactions.** Composition reactions (`@Async @TransactionalEventListener`) run after commit, one listener method per target module. `EventScopes.run` sets the event's tenant before the listener's own transaction starts.
- **Redelivery.** Outstanding publications are redelivered on restart.
- **Idempotent consumers.** Ledger entries are unique per `(source_type, source_uid)`. Reporting keeps a `processed_events` guard.

## Consequences
- Nothing is published for a write that rolled back. Downstream failures don't fail the sale.
- Read models and the ledger are eventually consistent, typically within milliseconds.
- Consumers must stay idempotent.
