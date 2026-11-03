# 0001 Modular monolith with Spring Modulith

- Status: accepted
- Date: 2026-11-02

## Context
The product needs strong domain boundaries (sales, inventory, finance, …) and has to scale. The team is small and wants one deployment and one database to operate. Starting with microservices would add network failure modes, distributed transactions and operational cost before they are needed.

## Decision
Build one deployable modular monolith:
- **Maven.** Each domain module is its own Maven module that depends on `pharmacy-platform` only, so `javac` rejects cross-module imports.
- **Ports.** Modules expose `ports.inbound`, `ports.outbound` and `events` as Spring Modulith named interfaces.
- **Where modules meet.**
  - Application services (onboarding, checkout, receiving) orchestrate writes that span modules.
  - `delivery` is the HTTP edge.
  - `composition` binds outbound ports to inbound ports and turns events into calls.
- **Verification.** `ApplicationModules.verify()` and ArchUnit rules run in the build.

## Consequences
- In-process calls and local transactions. One artifact to deploy.
- A module can later be extracted as a service. Its schema, ports and events are already its boundary.
- Compile-time enforcement needs more POMs and an explicit composition layer.
