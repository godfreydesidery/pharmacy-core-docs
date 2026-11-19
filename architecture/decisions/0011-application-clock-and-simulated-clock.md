# 0011 One application clock, and a simulated clock for simulated worlds

- Status: accepted
- Date: 2026-11-19

## Context
Much of the product depends on the date:
- trials, grace periods and suspension;
- receivable and claim ageing, and insurer terms;
- expiry and FEFO;
- period locks, month-end counts and reports.

None of it could be tested over weeks or months. 48 places read the system time directly
(`Instant.now()`, `LocalDate.now()`, SQL `now()`), so the API could only run on today's date. We
also wanted a whole-market simulation (the `pharmacy-world` workspace) to play a quarter of
trading through the real API and check the books every night.

## Decision
- **One clock.**
  - All business time comes from one application `Clock`. Services inject it; entities and static
    helpers use `Clocks.now()`, which Spring points at the same bean.
  - SQL binds the application's time instead of `now()`, and audit partitions are created from the
    clock's day.
  - An ArchUnit rule forbids direct `now()` calls. The exceptions are uid ordering, TOTP and edge
    rate limits, which measure real time.
- **A simulated clock behind a profile.**
  - Under the `sim` profile the clock is a `SimulatedClock`: it starts at a chosen instant, runs at
    real speed, only moves forward, and persists its position across restarts.
  - A root-only route (`/api/v1/platform/sim/clock`) moves it. The route exists only under `sim`.
  - The same profile lengthens sessions, lifts the sign-in limit, turns off the demo seed and the
    rate refresh, and makes the payment simulator decline by phone number.
- **A guard.** The API refuses to start under `sim` unless the `local` profile is also active, or
  `pharmacy.sim.allowed=true` (`PHARMACY_SIM_ALLOWED`) declares a dedicated demo service.

## Consequences
- Date logic is testable with a fixed or simulated clock, and a whole quarter runs in an afternoon.
  The first quarter found two real bugs, a batch-cost drift and a sign-in race.
- Normal deployments behave exactly as before. The clock is the system clock and the route does not
  exist.
- Moving the simulated clock is irreversible and affects every tenant on that server, so `sim` must
  never run where real pharmacies trade. The guard makes that a startup failure rather than a
  convention.
- Scheduled jobs still fire on real-time cron. A simulation triggers the daily work itself.
- New code that needs the time must take the `Clock` (or use `Clocks.now()`); the build enforces it.
