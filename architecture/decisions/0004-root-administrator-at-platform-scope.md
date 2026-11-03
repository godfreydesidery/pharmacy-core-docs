# 0004 Root administrator at platform scope

- Status: accepted
- Date: 2026-11-02

## Context
The platform operator must onboard and suspend pharmacies and support any of them, without being a member of each one.

## Decision
- **The flag.** `access.users.root_admin` marks a root administrator.
- **Platform routes.** Root administrators use `/api/v1/platform/*`: tenants, root administrators and the overview.
- **Acting in a tenant.** They may name any tenant and branch, and `ExecutionContext.Frame.rootAdmin` holds every permission. Their actions are recorded under their own identity (`created_by` / `updated_by` = their email).
- **Bootstrap.** The first root administrator is created at start from `PHARMACY_ROOT_EMAIL` / `PHARMACY_ROOT_PASSWORD` when none exists.

## Consequences
- One accountable identity for platform support, with no hidden memberships.
- Root sessions are powerful. Enabling 2FA for root administrators is the next step.
