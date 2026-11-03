# Deployment

## Database roles (once per database)

```sql
create role pharmacy_owner login password '<owner-secret>';
create role pharmacy_app   login nosuperuser nobypassrls password '<app-secret>';
create database pharmacy owner pharmacy_owner;
```

- The API connects as `pharmacy_app`.
- It refuses to start if that role is a superuser, bypasses RLS, or is the owner.
- Migrations run as `pharmacy_owner`.

## Configuration

| Variable | Required | Notes |
|---|---|---|
| `PHARMACY_DB_URL` | yes | `jdbc:postgresql://host:5432/pharmacy` |
| `PHARMACY_DB_USER` / `PHARMACY_DB_PASSWORD` | yes | The application role |
| `PHARMACY_DB_OWNER_USER` / `PHARMACY_DB_OWNER_PASSWORD` | when migrating at start | The owner role |
| `PHARMACY_DB_MIGRATIONS_ENABLED` | no (`true`) | `false`: the API does not migrate and holds no owner password |
| `PHARMACY_DB_POOL_SIZE` | no (`20`) | Per instance. Keep the sum across instances below PostgreSQL `max_connections` |
| `PHARMACY_ROOT_EMAIL` / `PHARMACY_ROOT_PASSWORD` | first start | Creates the first root administrator if none exists. Change the password afterwards |
| `PHARMACY_CORS_ORIGINS` | if the web app is on another origin | Comma-separated |
| `PHARMACY_SESSION_HOURS` | no (`12`) | Session lifetime |
| `PHARMACY_SWAGGER_ENABLED` | no (`true`) | Set `false` in production if the docs must not be public |
| `PHARMACY_LOG_FORMAT` | no | `ecs` or `logstash` for JSON logs with `correlationId` |
| `PHARMACY_PAYMENTS_PROVIDER` / `PHARMACY_FISCAL_PROVIDER` | no (`manual` / `none`) | Select integration adapters |
| `PHARMACY_SEED_DEMO` | no (`false`) | Never `true` in production |

## Migrations as a separate step

1. Run a one-off container with the owner credentials and `PHARMACY_DB_MIGRATIONS_ENABLED=true`. Wait until it reports started, then stop it. A dedicated migrate-only mode is on the roadmap.
2. Run the API instances with `PHARMACY_DB_MIGRATIONS_ENABLED=false`, without owner credentials.

Migrations are additive (expand, migrate, contract), so the old and new versions can run side by side during a rolling deploy.

## Health and probes

| Probe | Path |
|---|---|
| Liveness | `/actuator/health/liveness` |
| Readiness | `/actuator/health/readiness` |
| Metrics | `/actuator/metrics` |

The API shuts down gracefully (`server.shutdown: graceful`). Undelivered events stay in `platform.event_publication` and are redelivered on the next start.

## Scaling

- **API.** It is stateless. Scale horizontally behind a load balancer, with at least 2 instances across availability zones. Sessions live in the database.
- **Database.** Primary plus read replicas. Partition the large append-only tables by month as they grow. Shard tenants by `tenant_uid` when one cluster isn't enough. See `architecture/overview.md` §7.
- **Background work.** Event listeners run on virtual threads inside each instance. A publication is delivered once per publishing instance and redelivered on restart.
