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
| `PHARMACY_ACCESS_ENCRYPTIONKEY` | **yes** | At least 32 random characters. Seals TOTP seeds and provider secrets. Changing it makes them unreadable (rotation is planned), so keep it in a secret store |
| `PHARMACY_FILES_SIGNING_SECRET` | **yes** with local file storage | At least 32 random characters. Signs file download and upload links |
| `PHARMACY_FILES_STORAGE` | no (`local`) | `local` (a directory) or `s3` (S3, MinIO, R2). Several instances need `s3` |
| `PHARMACY_FILES_ROOT` | **yes** with local storage | An absolute path on a volume, e.g. `/data/files` |
| `PHARMACY_ACCESS_WEBURL` | **yes** | The web app's public address; reset and invitation links point there |
| `PHARMACY_REQUIRE_2FA_FOR_ROOT` | no (`true`) | Root administrators must add an authenticator app at their next sign-in |
| `PHARMACY_SESSION_HOURS` | no (`12`) | Session lifetime |
| `PHARMACY_SWAGGER_ENABLED` | no (`false`) | `true` publishes `/swagger-ui.html` and `/v3/api-docs`; leave off on a server |
| `PHARMACY_LOG_FORMAT` | no | `ecs` or `logstash` for JSON logs with `correlationId` |
| `PHARMACY_PAYMENTS_PROVIDER` / `PHARMACY_FISCAL_PROVIDER` | no (`manual` / `none`) | Select integration adapters |
| `PHARMACY_SEED_DEMO` | no (`false`) | Never `true` in production |
| `SPRING_PROFILES_ACTIVE` | no | Never include `sim` on a server with real pharmacies; see below |
| `PHARMACY_SIM_ALLOWED` | no (`false`) | Only for a dedicated demo service that runs the `sim` profile on its own database |

> **The API refuses to start on development defaults.** Outside the `local` and `test` profiles,
> start-up stops (before any migration) when the encryption key or the file signing secret is
> missing, is the published default or is shorter than 32 characters, when the file root is a
> relative path, or when `PHARMACY_ACCESS_WEBURL` is unset or points to localhost. The log names
> each variable. The `log` message providers (used when no provider is configured for a channel)
> then write only a masked address and the message length, never the body: a message may carry a
> reset link or a sign-in code.

> **The `sim` profile is for simulations only.** It replaces the system clock with a simulated
> clock that moves every pharmacy's time forward for good, and it relaxes sessions, sign-in limits
> and payments. The API refuses to start under `sim` unless the `local` profile is also active, or
> `PHARMACY_SIM_ALLOWED=true` declares a dedicated demo service. See
> [simulated-world.md](simulated-world.md).

## Migrations as a separate step

1. Run a one-off container with the owner credentials and `PHARMACY_DB_MIGRATIONS_ENABLED=true`. Wait until it reports started, then stop it. A dedicated migrate-only mode is on the roadmap.
2. Run the API instances with `PHARMACY_DB_MIGRATIONS_ENABLED=false`, without owner credentials.

Migrations are additive (expand, migrate, contract), so the old and new versions can run side by side during a rolling deploy.

## Health and probes

| Probe | Path |
|---|---|
| Liveness | `/actuator/health/liveness` |
| Readiness | `/actuator/health/readiness` |

Only `health` is exposed over HTTP. Metrics are not public; they will be scraped on a private
port when monitoring is added.

The API shuts down gracefully (`server.shutdown: graceful`). Undelivered events stay in `platform.event_publication` and are redelivered on the next start.

## Scaling

- **API.** Sessions live in the database, but several instances are **not yet safe**: scheduled
  jobs (webhook retries, subscriptions, retention, digests) run on every instance, sign-in limits,
  the per-pharmacy request cap, provider health and live notifications are kept in each instance's
  memory, and local file storage is one volume. Run **one** instance until the multi-instance work
  (adversarial review 23 Nov 2026, item 7 of the owner's answers) is done; the target is several
  instances (for example ECS tasks) behind a load balancer.
- **Database.** Primary plus read replicas. Partition the large append-only tables by month as they grow. Shard tenants by `tenant_uid` when one cluster isn't enough. See `architecture/overview.md` §7.
- **Background work.** Event listeners run on virtual threads inside each instance. A publication is delivered once per publishing instance and redelivered on restart.
