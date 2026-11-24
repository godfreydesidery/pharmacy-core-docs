# 0002 PostgreSQL with row-level security for tenant isolation

- Status: accepted
- Date: 2026-11-02

## Context
Many pharmacies share one database. A leak between tenants is the worst failure the system can have. With application-only filtering, a single forgotten `WHERE tenant_uid = ?` is a breach. MySQL was considered. It has no row-level security, so isolation would rest on application code alone.

## Decision
Use PostgreSQL (17 when decided; 18 now, see ADR 0010):
- **Tenant column.** Every tenant-owned table has `tenant_uid`.
- **Policy.** Each such table has a forced policy: `using/with check (tenant_uid = platform.current_tenant_uid())`.
- **Setting the tenant.** `TenantAwareTransactionManager` sets `pharmacy.tenant_uid` (and `pharmacy.user_uid`) with `set_config(..., true)` at the start of every transaction.
- **Roles.** The API connects as `pharmacy_app` (`nosuperuser nobypassrls`, owns nothing). Migrations run as `pharmacy_owner`.
- **Startup check.** The API refuses to start with a privileged role.

## Consequences
- **Fails closed.** With no tenant set, tenant tables look empty and inserts are refused.
- **Bonus features.** We also get transactional DDL, partitioning, `jsonb`, partial indexes and CHECK constraints.
- **Cross-tenant reads.** Platform-wide reads (the tenant registry, user identities) need tables without RLS, or a `SECURITY DEFINER` function. These are kept few and explicit.
- **Tests.** Tests must run against PostgreSQL with the same roles. They use an embedded PostgreSQL.
