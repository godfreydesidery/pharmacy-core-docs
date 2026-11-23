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
| `PHARMACY_DB_OWNER_USER` / `PHARMACY_DB_OWNER_PASSWORD` | when migrating | The owner role; only the migrate step holds it |
| `PHARMACY_MIGRATIONS_MODE` | no (`migrate-and-run`) | `migrate-only` (migrate, exit 0/1) or `run-only` (no owner password; refuses to start with a pending migration). `PHARMACY_DB_MIGRATIONS_ENABLED=false` still means `run-only` |
| `PHARMACY_MIGRATIONS_LOCK_TIMEOUT` / `_STATEMENT_TIMEOUT` | no (`5s` / `5min`) | A migration waiting on a lock fails fast and rolls back |
| `PHARMACY_DB_POOL_SIZE` | no (`20`) | Per instance. Keep the sum across instances below PostgreSQL `max_connections` |
| `PHARMACY_ROOT_EMAIL` / `PHARMACY_ROOT_PASSWORD` | first start | Creates the first root administrator if none exists. Change the password afterwards |
| `PHARMACY_CORS_ORIGINS` | if the web app is on another origin | Comma-separated |
| `PHARMACY_ACCESS_ENCRYPTIONKEY` | **yes** until removable | The `legacy` key of the keyring, at least 32 random characters. Keep it until `GET /api/v1/platform/keys` shows it removable |
| `PHARMACY_KEYS_<ID>` | no | Named keys of the keyring (lower-case id, at least 32 random characters) |
| `PHARMACY_SEALING_TOTP` / `_PROVIDERS` / `_WEBHOOKS` | no (`legacy`) | The active key per purpose. Rotation: api repo `docs/runbook/rotating-keys.md` |
| `PHARMACY_FILES_SIGNING_SECRET` | **yes** | At least 32 random characters. Signs the upload links (both stores) and local download links |
| `PHARMACY_FILES_STORAGE` | no (`local`) | `local` (a directory, one instance) or `s3` (S3, MinIO, R2; any number of instances). Cut-over: api repo `docs/runbook/files-on-s3.md` |
| `PHARMACY_FILES_S3_BUCKET` / `_REGION` / `_ENDPOINT` / `_PATH_STYLE` / `_PREFIX` / `_SSE` / `_KMS_KEY_ID` | with `s3` | Credentials come from the AWS default chain (the ECS task role), never from variables |
| `PHARMACY_FILES_ROOT` | **yes** with local storage | An absolute path on a volume, e.g. `/data/files` |
| `PHARMACY_ACCESS_WEBURL` | **yes** | The web app's public address; reset and invitation links point there |
| `PHARMACY_REQUIRE_2FA_FOR_ROOT` | no (`true`) | Root administrators must add an authenticator app at their next sign-in |
| `PHARMACY_SESSION_HOURS` | no (`12`) | Session lifetime |
| `PHARMACY_SWAGGER_ENABLED` | no (`false`) | `true` publishes `/swagger-ui.html` and `/v3/api-docs`; leave off on a server |
| `PHARMACY_LOG_FORMAT` | no | `json`/`ecs`, `logstash` or `gelf` for JSON logs with `correlationId` |
| `PHARMACY_MANAGEMENT_PORT` | no | A private actuator port (health, `/actuator/prometheus`), e.g. `8081` on ECS; `/livez` and `/readyz` stay on the API port |
| `PHARMACY_INSTANCE_ID` / `PHARMACY_JOB_RUNS_DAYS` / `PHARMACY_EVENTS_REPUBLISH_ON_START` | no | Instance name in job runs; days of job runs kept (30); keep republishing at start `false` with several instances |
| `SENTRY_DSN` / `SENTRY_ENVIRONMENT` | no | Optional error tracker; off when unset, no personal data sent |
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

1. Run the API image once with `PHARMACY_MIGRATIONS_MODE=migrate-only` and the owner credentials. It
   migrates every schema and exits 0 (1 on failure); nothing else starts.
2. Only after exit code 0, roll the API out with `PHARMACY_MIGRATIONS_MODE=run-only` and no owner
   credentials. It refuses to start while a migration is pending.

Migrations are additive (expand, migrate, contract), so the old and new versions run side by side
during a rolling deploy; a `run-only` instance accepts migrations newer than itself. Each migration
statement waits at most 5 s for a lock. What a partly finished run leaves and how to resume: api
repo `docs/runbook/migrations.md`. Railway still uses `migrate-and-run`.

## Health and probes

| Probe | Path |
|---|---|
| Liveness | `/livez` on the API port (and `/actuator/health/liveness`) |
| Readiness | `/readyz` on the API port (and `/actuator/health/readiness`) |

With `PHARMACY_MANAGEMENT_PORT` set, actuator (health groups and `/actuator/prometheus`) moves to
that private port; `/livez` and `/readyz` stay on the API port for the load balancer and the
container check. Without it, actuator shares the API port and serves only health.

The API shuts down gracefully (`server.shutdown: graceful`). Undelivered events stay in
`platform.event_publication` and the `event-resubmission` job delivers them again.

## Backups and restore

Production backups (RDS point-in-time recovery, snapshots, a portable encrypted `pg_dump` to S3),
the restore procedure with its checks, RPO 5 min / RTO 1 h and the quarterly drill: api repo
`docs/runbook/backup-restore.md` (Drill 1 restored the QA copy: 125 tables, every count equal).

## Scaling

- **API.** Several instances are safe (Batch 6): see "Running several instances" below.
- **Database.** Primary plus read replicas. Partition the large append-only tables by month as they grow. Shard tenants by `tenant_uid` when one cluster isn't enough. See `architecture/overview.md` §7.
- **Background work.** Event listeners run on virtual threads inside each instance. An undelivered publication is delivered again by the `event-resubmission` job on one instance.

## Running several instances

The API runs as several instances at once (for example two or more ECS Fargate tasks behind an
ALB). Nothing a request depends on is kept in one instance's memory: every instance reads and
writes the same PostgreSQL, so any instance can serve any request, and no sticky sessions are
needed.

### Shared through PostgreSQL

| What | How |
|---|---|
| Sessions, account locks | Rows in `access` (as before) |
| Scheduled jobs | Each job takes a PostgreSQL advisory lock named after it (`SingleRunner`, `pg_try_advisory_lock`); an instance that finds it taken skips the round. The lock goes with its connection, so a crashed task never leaves it held |
| Queue-like work | Rows are claimed by one worker: exports `FOR UPDATE SKIP LOCKED` with heartbeats; webhook deliveries `FOR UPDATE SKIP LOCKED` with a lease; notification messages by a conditional update one caller wins (with a lease); the SMS cap under a transaction advisory lock |
| Job status | `platform.job_runs`: job, instance, start, end, outcome (OK, FAILED, SKIPPED), message, the pharmacies a run failed for. Root reads it at `GET /api/v1/platform/jobs` (and `/jobs/{job}/runs`). 30 days are kept (`PHARMACY_JOB_RUNS_DAYS`) |
| Sign-in limits | Failures per client IP (20 in 15 min, then 15 min locked) and per unknown email (5 in 15 min), and the sign-in requests per IP (20 a minute): a sliding window per key in `platform.rate_windows`, updated by one atomic upsert per hit. Keys are stored as SHA-256 hashes (no email or IP address). Expired rows go with the nightly retention |
| Root activity notes | The "root acted in this pharmacy" audit entry, once an hour per administrator and pharmacy, counted in `platform.rate_windows` |
| Live notifications (SSE) | A new in-app message is signalled with `NOTIFY pharmacy_inapp` (payload: instance, pharmacy and user ids, no personal data). Each instance holds **one dedicated listener connection** (outside the pool, `application_name = pharmacy-signals`); the instance that holds the user's stream reads the message and the unread count from the database and pushes them. After the listener reconnects (database restart, failover), every open stream gets its current state again |
| Undelivered internal events | Delivered again by the `event-resubmission` job (every 5 min, older than 5 min, one instance). Republishing at start-up is off (`PHARMACY_EVENTS_REPUBLISH_ON_START=false`), because a starting instance would deliver again the events the others are still delivering |
| Files | `PHARMACY_FILES_STORAGE=s3` (one bucket for all instances). Local storage is one volume and does not work with several instances |

### Per instance, on purpose

| What | Why it stays per instance |
|---|---|
| The SSE connections themselves | A stream is a TCP connection to one task; the fan-out above reaches it from any instance |
| Fair share per pharmacy (`PHARMACY_TENANT_CONCURRENCY`, 10 requests at once) | It shares out **this instance's** database pool; with N instances a pharmacy may have N × 10 in progress, which N pools can serve |
| Argon2 hashing slots | They protect this instance's CPU and memory (19 MiB per hash) |
| Provider circuit breaker (`ProviderHealth`) | It only orders the providers to try; a tripped provider is still tried. Each instance learns after 3 failures of its own. The provider's last success and error are on its row, shared |
| "Worked in this pharmacy" write throttle | Saves writes only; the one SIGN_IN audit entry per session and pharmacy is decided in the database |
| Short caches | A pharmacy's 2FA policy (30 s), time zone, the SMTP sender built from a provider row |
| Stream heartbeats (25 s) | Each instance pings its own streams |
| The export worker's poll (5 s) | Every instance polls; the claim makes each export one instance's |

### Scheduled jobs

| Job (`platform.job_runs.job`) | When | Runs on |
|---|---|---|
| `retention` | 01:40 UTC | one instance (advisory lock) |
| `subscription-lifecycle` | 02:15 | one instance |
| `audit-partitions` | start and 02:15 UTC | one instance |
| `fx-refresh` | 06:00 UTC | one instance |
| `notification-digests` | every 15 min | one instance |
| `notification-retries` | every minute | one instance; each message claimed once |
| `webhook-retries` | every minute | one instance; each delivery claimed once |
| `event-resubmission` | every 5 min | one instance |
| `data-export` | each export built | the instance that claimed it |
| stream heartbeats | every 25 s | every instance (its own streams; not recorded) |

A failed run (an exception, or a pharmacy the run failed for) is recorded FAILED, counted, and
logged at ERROR with the stable prefix **`JOB_FAILED`**:
`JOB_FAILED job=<job> instance=<instance> took=<ms>ms failures=<n> message=<...> failed for: <pharmacy> (<error>) ...`

### What AWS must provide

**Ports and the load balancer**

- Container port **8080**: the API, the ALB target group.
- Container port **8081**: actuator (health and Prometheus metrics), set with
  `PHARMACY_MANAGEMENT_PORT=8081`. Do not add it to the ALB listener; allow it in the task's
  security group only from inside the VPC (the metrics collector).
- ALB target group health check: `GET /readyz` on the traffic port (8080), healthy on 200. (On the
  management port the same check is `/actuator/health/readiness`.)
- ECS container health check (in the task definition, not the image's `HEALTHCHECK`):
  `CMD-SHELL wget -qO- http://localhost:8081/actuator/health/liveness || exit 1`, start period
  60–120 s (migrations run before readiness when they run at start).
- ALB idle timeout at least 60 s (the default) so SSE streams with a 25 s heartbeat stay open;
  deregistration delay of 30–60 s (the API shuts down gracefully).
- No sticky sessions.

**Environment**

| Variable | Value on ECS | Notes |
|---|---|---|
| `PHARMACY_MANAGEMENT_PORT` | `8081` | Unset: actuator shares 8080 and serves only health (local, Railway) |
| `PHARMACY_LOG_FORMAT` | `json` | One JSON object per line on stdout (Spring Boot structured logging, Elastic Common Schema: `@timestamp`, `log.level`, `log.logger`, `message`, `correlationId`, `error.*`). `ecs`, `logstash`, `gelf` also work; unset is plain text |
| `PHARMACY_FILES_STORAGE` | `s3` | With `PHARMACY_FILES_S3_BUCKET` and `PHARMACY_FILES_S3_REGION=af-south-1`; the task role grants the bucket |
| `PHARMACY_DB_POOL_SIZE` | e.g. `20` | Per instance. Keep N × (pool + 1 listener) + the migrator below RDS `max_connections` |
| `PHARMACY_INSTANCE_ID` | optional | Defaults to the host name (the task's) and a random suffix; shown in job runs |
| `SENTRY_DSN` | optional | The error tracker (Sentry). Off when unset; no personal data is sent (`send-default-pii=false`, no request bodies). `SENTRY_ENVIRONMENT` tells servers apart |
| `PHARMACY_JOB_RUNS_DAYS` | `30` | How long job runs are kept |
| `PHARMACY_EVENTS_REPUBLISH_ON_START` | `false` | Keep false with several instances |

**Database (RDS PostgreSQL)**

- Point `PHARMACY_DB_URL` at the RDS instance or cluster endpoint, **not RDS Proxy**: the job
  locks (session advisory locks) and the listener (`LISTEN`) need a database session of their own,
  which RDS Proxy pins or does not support.
- `LISTEN/NOTIFY` works on RDS PostgreSQL as is; nothing to enable.

**Metrics and alarms**

- Scrape `http://<task>:8081/actuator/prometheus` with the ADOT collector (a sidecar in the task
  scraping `localhost:8081`, or a collector service in the VPC), exporting to Amazon Managed
  Service for Prometheus or to CloudWatch (EMF exporter).
- Metrics (Prometheus names):
  - jobs: `pharmacy_jobs_runs_total{job,outcome}`, `pharmacy_jobs_duration_seconds{job,outcome}`,
    `pharmacy_jobs_failures_total{job}`;
  - sign-in: `pharmacy_signin_failures_total{kind=account|unknown_email}`,
    `pharmacy_signin_locks_total{kind=account|unknown_email|client}`;
  - notifications: `pharmacy_notifications_queued_total{channel}`, `..._sent_total{channel}`,
    `..._failed_total{channel}`;
  - webhooks: `pharmacy_webhooks_deliveries_total{outcome=delivered|failed}`;
  - exports: `pharmacy_exports_total{outcome=ready|retried|failed|abandoned}`;
  - HTTP and the platform (built in): `http_server_requests_seconds`, `hikaricp_connections_*`,
    `jvm_*`, `process_*`.
- Alarms:
  - **a job failed**: a CloudWatch Logs metric filter on the API's log group,
    `{ $.message = "JOB_FAILED*" }` (JSON logs) or the term `"JOB_FAILED"` (plain text), alarm on
    ≥ 1 in 5 minutes; or `increase(pharmacy_jobs_runs_total{outcome="FAILED"}[5m]) > 0`;
  - **a nightly job did not run**: `sum(increase(pharmacy_jobs_runs_total{job="retention",outcome="OK"}[26h])) == 0`
    (likewise `subscription-lifecycle`); root sees the same at `GET /api/v1/platform/jobs`;
  - lock-outs: `increase(pharmacy_signin_locks_total{kind="client"}[15m])` above normal;
  - outbound: `pharmacy_notifications_failed_total`, `pharmacy_webhooks_deliveries_total{outcome="failed"}`;
  - errors: 5xx rate from `http_server_requests_seconds_count{status=~"5.."}` (or the ALB's own
    `HTTPCode_Target_5XX_Count`).
- Traces are not emitted by the API. If they are wanted, add the OpenTelemetry Java agent to the
  task (`JAVA_TOOL_OPTIONS=-javaagent:...`) sending OTLP to the same ADOT collector; no code change.
