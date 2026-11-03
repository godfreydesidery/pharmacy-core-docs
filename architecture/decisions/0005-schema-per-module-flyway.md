# 0005 Schema per module, migrated by its own Flyway

- Status: accepted
- Date: 2026-11-02

## Context
Module boundaries should hold in the database too, so that a module can leave with its data.

## Decision
- **One schema per module.** Each module owns one schema: `tenancy`, `access`, `catalogue`, … Shared infrastructure sits in `platform`: helper functions, the event outbox and document sequences.
- **Migrations travel with the module.** Each module's migrations live in its own jar (`db/migration/<schema>`).
- **Separate runs.** `ModuleMigrations` runs one Flyway per schema, in a fixed order, with that schema's own history table, as the owner role. Grants go to `${application_role}`.
- **Before Hibernate.** Migrations run before Hibernate validates the schema (`ddl-auto: validate`). With `PHARMACY_DB_MIGRATIONS_ENABLED=false` they run as a separate step instead.

## Consequences
- No module reads another's tables. Data crosses modules only through ports and events.
- Migrations are append-only. A change is a new version.
