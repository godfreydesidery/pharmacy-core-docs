# 0007 FEFO allocation with row locks

- Status: accepted
- Date: 2026-11-02

## Context
Pharmacies must sell the batch that expires first, and must never sell an expired batch. Concurrent tills at one branch must not oversell.

## Decision
- **Where.** `inventory.StockService.allocate` runs inside the checkout transaction.
- **Which rows.** It locks the branch's stock rows for the product (`PESSIMISTIC_WRITE`, i.e. `SELECT … FOR UPDATE`), with quantity above 0 and `expiry_date >= today`, ordered by expiry date.
- **What it does.** It takes from the earliest-expiring batch first and records a `SALE` movement per batch. It refuses with `insufficient_stock` when the stock is short.
- **The ledger.** Every stock change is an append-only movement with `balanceAfter`.

## Consequences
- Correct under concurrency. Locks are held per (branch, product), so different products and branches don't contend.
- A sale line may split across batches. Each becomes its own sale line with its batch and expiry, which also gives traceability for recalls.
