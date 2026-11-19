# Simulated clock and the simulated world

Pharmacy Core can run on a **simulated clock** that moves forward on command. On that clock, the
separate **`pharmacy-world`** workspace plays a whole market through the real API: about 50
Tanzanian pharmacies trading through a business quarter in an afternoon. Every night it checks the
books against what the world actually did. Use it for QA and regression, and to test anything that
depends on dates: trials, grace periods, ageing, expiry and month ends.

Why it is built this way: [ADR 0011](../architecture/decisions/0011-application-clock-and-simulated-clock.md).

> **Never enable the `sim` profile where real pharmacies trade.** The simulated clock moves *every*
> pharmacy's time forward for good, and the profile relaxes sessions, sign-in limits and payments.
> The API refuses to start under `sim` unless the `local` profile is also active, or
> `PHARMACY_SIM_ALLOWED=true` marks a dedicated demo service.

## How time works in the API

| Where the API runs | Clock | What is different |
|---|---|---|
| Normal deployment (Railway, production) | The system clock, in UTC | Nothing |
| Under the `sim` profile | A `SimulatedClock`: runs at real speed from a chosen start and moves forward on command | See *The `sim` profile* below |

All code reads the time from one application clock: services take the injected `Clock`, and
entities and static helpers use `Clocks.now()`. An architecture test
(`CodeRulesTest.timeComesFromTheClock`) fails the build if anything calls `Instant.now()`,
`LocalDate.now()` and the like directly. Three things deliberately stay on wall time:

- uid generation (ordering);
- TOTP checks (codes come from the user's phone);
- the edge rate limits (they measure real traffic).

Audit entries are partitioned by month. The nightly job creates partitions from the clock's day
(audit migration V3), so a simulated month always has its own partition.

## The `sim` profile

Turn it on beside `local`: `--spring.profiles.active=local,sim` (or
`SPRING_PROFILES_ACTIVE=local,sim`).

| Setting | Value under `sim` | Why |
|---|---|---|
| Clock | `SimulatedClock`, starting at `pharmacy.sim.start` (ISO instant) or now | The world chooses its dates |
| `pharmacy.sim.clock-file` | `target/sim-clock.txt` by default | A restart resumes where the world was |
| Session lifetime | 2400 hours | Simulated staff do not sign in again every simulated day |
| Sign-in rate limit | 100,000 a minute | One machine plays hundreds of staff |
| Demo seed (Afya) | off | The world seeds its own pharmacies |
| Exchange-rate refresh | off | Nothing fetches from outside |
| Payment simulator | answers by phone number (table below) | Push payments can fail as in real life |

### Moving the clock

Root administrator only. The route exists only under `sim`.

```http
GET  /api/v1/platform/sim/clock                                  → { "now": "2027-01-15T05:00:00Z" }
POST /api/v1/platform/sim/clock  { "advance": "P1D" }            → moves forward one day (ISO duration)
POST /api/v1/platform/sim/clock  { "to": "2027-02-01T05:00:00Z" } → moves forward to that instant
```

The clock only moves forward. Asking for an earlier moment returns `422 clock_backwards`, because
data written at a simulated moment must never appear to come from the future. Each move also makes
sure the audit partitions exist for the new month.

### Payment simulator outcomes

The `SIMULATOR` payment method is switched off on the platform by default; the world switches it on.
Under `sim` (`pharmacy.integrations.payments.simulator-outcomes=by-phone`), the last two digits of
the customer's phone decide what happens:

| Phone ends in | Outcome |
|---|---|
| `01` | The customer declines (`FAILED`) |
| `02` | Insufficient funds (`FAILED`) |
| `03` | No answer; the request stays `PENDING` and expires as simulated time passes |
| anything else | Approved (`CONFIRMED`) |

Outside `sim`, the simulator approves every request. The real M-Pesa, Mixx, Airtel, Selcom and
AzamPay adapters are not built yet. Cashiers enter the provider's confirmation code by hand, and
the world does the same.

## Running the local world

The world lives in its own workspace, `pharmacy-world`, next to the three repositories. It needs
Node 22+, Java 21, Maven, Docker Desktop and an API jar built from a branch that has the simulated
clock:

```bash
cd pharmacy-core-api
mvn -Dpharmacy.build.dir=target-mvn package -DskipTests

cd ../pharmacy-world
npm install
npm run up                       # Mailpit + the API (local,sim) on the world's own database and clock
npm run world -- run             # plays the quarter (1 Dec 2026 – 28 Feb 2027); resumes after a stop
npm run world -- report          # .data/report.html from the API and the run's history
npm run world -- watch           # the web app on http://localhost:4390 against the world
npm run down                     # stops the API (database first), Mailpit and the web app
```

| Command | What it does |
|---|---|
| `run --days 7 --only UZIMA,NEEMA` | A week, two pharmacies |
| `run --traffic 0.5` | Half the customers (faster) |
| `run --days 1 --pace 10 --only UZIMA` | One day, ten real seconds per simulated hour, for watching in the web app |
| `touch .data/stop` | Stops cleanly after the day under way |
| `clock` / `clock P1D` | Shows or advances the simulated time |
| `reset --yes` | Deletes the world (database, files, clock, history) |

The world never shares a port or a database with the demo run (8080/54330) or the QA copy:

| What | Where |
|---|---|
| API | `http://localhost:8090/api/v1` |
| Database | embedded PostgreSQL on port 54340, data in `pharmacy-world/.data/postgres` |
| Every email the world sends | Mailpit, `http://localhost:8025`. SMS and WhatsApp use the `log` provider. |
| Web app (watch mode) | `http://localhost:4390` |
| Root administrator | `root@pharmacy.local` / `Root#2026` |
| Every staff member | password `World#2026pass`; `watch` lists the focal pharmacies' owners and cashiers |

A full quarter of 50 pharmacies takes about 1.5 hours on a laptop. Every decision draws from a
seeded random stream (seed `tz-2026`), so the same seed plays the same market.

### What the world contains

- **Pharmacies.** 5 focal pharmacies, simulated in full detail:
  - a 6-branch Dar es Salaam chain on Pro;
  - a 3-branch Arusha/Moshi pharmacy on Standard;
  - a busy Mwanza shop on Starter;
  - an insurance-heavy hospital pharmacy in Dodoma;
  - a Tanga shop that never pays its subscription.

  45 background pharmacies across 15 regions join over the first five weeks.
- **A simulated day.**
  - 07:00 EAT: new pharmacies join and transfers arrive.
  - 08:00: tills open.
  - Every hour until 20:00: customers buy. They pay by cash, M-Pesa, Mixx or Airtel (by reference),
    push payment, card, insurance or on account.
  - 20:00: tills close with a cash count.
  - 21:00, the back office: deliveries (some late or short), expiries, transfers, reorders, supplier
    payments, expenses, collections, claim batches, adjudication and remittances, month-end counts
    and period locks.
  - 23:00: the platform runs its subscription lifecycle, and pharmacies pay their invoices, pay
    late, or never pay.
- **Demand.** Each region has its own weight. The lake and coast regions buy more antimalarials in
  the December rains, Sundays are quiet, and paydays are busy. Wholesale prices rise 4% on 15 Jan.

### What the nightly checks prove

Every night for the focal pharmacies, and every Sunday and month end for all of them:

| Check | Compares |
|---|---|
| Ledger balanced | Debits with credits (`/accounts`) |
| Stock matches the world | Every shelf in the API with the world's own record of what it moved |
| Stock never negative | Every shelf |
| Inventory matches stock | Ledger account 1200 with the stock at cost (`/stock`) |
| Receivables | Ledger 1100 with what customers owe (`/receivables/totals`) |
| Payables | Ledger 2000 with what is owed to suppliers (`/suppliers/balances`) |
| Insurance | Ledger 1110 with the open claims |
| Sales report | `/reports/sales` for the day with the sales made, net of the day's returns |

Other checks run inline as the world works:

- **At each till close:** the API's expected cash equals the cash the till took, and the variance
  equals counted minus expected.
- **The day after each period lock:** a back-dated expense is refused.
- **Every night:** a pharmacy is suspended only after its grace period runs out unpaid, and
  paying what it owes brings it back.

A failure is written to `pharmacy-world/.data/findings.jsonl` and shown in the report. When a check
fails, find the root cause before trusting the count. One bug can fail hundreds of nightly checks.

### Bugs the world has found

| Found | Bug | Fixed in |
|---|---|---|
| Quarter 1 (Jan 2027 sim) | Receiving more of a batch at a new cost kept the old cost; the ledger's inventory drifted above stock at cost | `pharmacy-core-api` PR #6: weighted-average batch cost |
| Quarter 1 (Dec 2026 sim) | Simultaneous sign-ins of one person returned 409 `stale_version` | `pharmacy-core-api` PR #6: the sign-in update skips the version check |

### Gotchas

- **Plan-gated reports.** The trial balance, stock valuation and balance sheet are Pro-only. The
  checks use `/accounts` and `/stock`, which every plan can read.
- **Scheduled jobs.** The subscription lifecycle, audit partitions and digests still fire on
  real-time cron. The world triggers the daily work itself (`POST /platform/subscriptions/run`
  with `asOf`).
- **Batches.** A batch number has one expiry date, so the world names manufacturer lots by month.
- **Phones.** They must be in international form (`+255…`).
- **List size.** List pages hold at most 100 items.
- **The web app's date.** "Today" is the browser's real date. Move the date pickers into the
  simulated quarter.
- **Restarting mid-day.** A day interrupted mid-way is replayed. That day's sales-report check is
  skipped, because the report also holds the first attempt's sales.

## A dedicated demo service (optional)

To fast-forward a pre-filled pharmacy in front of a client, run a **separate** service with its own
database:

```bash
SPRING_PROFILES_ACTIVE=sim
PHARMACY_SIM_ALLOWED=true          # without it the API refuses to start under sim
PHARMACY_SIM_START=2027-01-01T05:00:00Z
PHARMACY_SIM_CLOCKFILE=/data/sim-clock.txt
```

Root moves the clock (for example through month end, a trial running out, or claims ageing), and
everyone on that service sees the new date. Never point it at a database with real pharmacies, and
never set `PHARMACY_SIM_ALLOWED` on the main service.
