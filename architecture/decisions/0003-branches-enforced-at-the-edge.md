# 0003 Branches enforced at the edge from the membership

- Status: accepted
- Date: 2026-11-02

## Context
A tenant has many branches. Some staff may use all of them, others only one. Reports may cover "all my branches". Enforcing branches in RLS as well would make cross-branch reports and transfers awkward and add a second session setting per query.

## Decision
- **Per request.** A request names its branch with `X-Branch-Id`.
- **Edge check.** `EdgeFilter` checks that the branch belongs to the tenant, is active, and is allowed by the membership (`allBranches` or `branchUids`). It then puts the branch into `ExecutionContext`.
- **Route scopes.** Branch routes are `@Access(scope = BRANCH)`. Reports use `TENANT_OPTIONAL_BRANCH` and `ExecutionContext.reportBranches()`: the named branch, the allowed set, or `null` for all.
- **Data.** Branch-owned rows carry `branch_uid`. Modules filter by it.

## Consequences
- The rule is simple and central. Transfers and reports across branches stay straightforward.
- Branch isolation is enforced in the application, not by the database. Tenant isolation is still enforced by the database.
