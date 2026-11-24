# pharmacy-core-docs

Documentation for **Pharmacy Core**, a multi-tenant, multi-branch pharmacy management system.

## The three repositories

| Repository | What | Stack |
|---|---|---|
| `pharmacy-core-api` | The backend: one modular monolith exposing `/api/v1` | Java 21 bytecode on a Java 25 runtime, Spring Boot 4.2, Spring Modulith 2.2, PostgreSQL 18 |
| `pharmacy-core-web` | The web app for the back office and POS | Angular 22, built and served with Node 26 |
| `pharmacy-core-docs` | This repository: product, contract, architecture, runbooks | Markdown, Mermaid |

The web app and any external system talk to the API only through the contract in `contracts/api-v1.md`.

## Contents

| Path | What |
|---|---|
| [`prd/pharmacy-prd.md`](prd/pharmacy-prd.md) | Product requirements (scope, NFRs, standards, release plan) |
| [`contracts/api-v1.md`](contracts/api-v1.md) | HTTP contract v1: headers, envelopes, permissions, every route and DTO |
| [`architecture/overview.md`](architecture/overview.md) | System and module architecture, request and event flows, tenancy, scaling |
| [`architecture/decisions/`](architecture/decisions/) | Architecture decision records (ADRs) |
| [`runbook/local-development.md`](runbook/local-development.md) | Run the API and the web app locally |
| [`runbook/deployment.md`](runbook/deployment.md) | Configuration, database roles, migrations, probes, scaling |
| [`runbook/simulated-world.md`](runbook/simulated-world.md) | The simulated clock (`sim` profile), and the simulated market that plays a quarter through the API and checks the books nightly |
| [`privacy/`](privacy/README.md) | Privacy pack (Tanzania PDPA 2022): records of processing, data map and transfers, sub-processors, breach response, DPO, data subject requests, retention, processor agreement outline |
| [`prototype/index.html`](prototype/index.html) | Clickable UI prototype (static, mock data) |
| [`presentations/`](presentations/) | Stakeholder deck (PDF and its HTML source) |

The binding module-level spec (ports, views, events) lives with the code, in `pharmacy-core-api/docs/MODULE-PORTS.md`.
