---
name: senior-dev
description: Senior-engineer rules for shipping production features the simplest, cleanest, safest way. Use this skill whenever building or reviewing anything that touches money, user accounts, data writes, background jobs, webhooks, external APIs, concurrency, or deployment — and especially for payments, checkout, subscriptions, refunds, credits/wallets, or anything where "what if two requests hit at once" or "what if the connection drops" matters. Use it even if the user just says "add Stripe", "build checkout", "make this production ready", or "ship this".
---

# Senior Dev

Ship the simplest thing that is **correct under failure**. Most bugs in production are not logic bugs — they are retries, races, partial failures, and assumptions about the network. Design for those first.

For any payment / billing / money work, **read `references/payments.md` before writing code.** For the final review before shipping, run through `references/preship-checklist.md`.

---

## 1. Core principles

1. **Simple beats clever.** Fewest moving parts, fewest services, fewest states. A boring Postgres table beats a queue + cache + microservice until proven otherwise.
2. **The database is the source of truth.** Not the client, not the cache, not memory, not the third-party dashboard. Every important fact has exactly one owner.
3. **Assume every network call can fail, hang, or succeed without you knowing.** The third case is the dangerous one.
4. **Make every write idempotent.** Running it twice must equal running it once.
5. **Let the database enforce invariants.** Unique constraints, foreign keys, CHECK constraints, and transactions catch what application code misses under concurrency.
6. **Never trust the client.** Prices, totals, roles, user IDs, plan names, discount amounts — recompute or look up on the server.
7. **Fail loudly in dev, gracefully in prod, and log everything in both.**
8. **Delete code before adding code.** Every line is a liability.
9. **Reversible decisions fast, irreversible decisions slow.** Schema changes, money movement, data deletion, and public APIs are irreversible — think first.

---

## 2. Concurrency rules

Two users, two tabs, two retries, or two workers will hit the same code at the same millisecond. Plan for it.

- **Check-then-act is a bug.** `if (!exists) insert()` races. Use a unique constraint + `INSERT ... ON CONFLICT DO NOTHING` (or catch the unique violation) instead.
- **Read-modify-write is a bug.** `balance = get(); set(balance - 10)` loses updates. Use atomic SQL: `UPDATE wallets SET balance = balance - 10 WHERE id = $1 AND balance >= 10 RETURNING balance`. Zero rows returned = insufficient funds.
- **Pick the lightest lock that works:**
  - Unique constraint → prevents duplicates (best, free).
  - Conditional `UPDATE ... WHERE status = 'pending'` → state transitions (optimistic, check row count).
  - `version` column → optimistic locking for edits.
  - `SELECT ... FOR UPDATE` inside a transaction → when you must read then decide.
  - `pg_advisory_xact_lock(hash)` → serialize work per user/resource without a row to lock.
  - Distributed locks (Redis etc.) → last resort; they expire and lie.
- **State machines, enforced in SQL.** Every status change is `UPDATE t SET status = 'b' WHERE id = $1 AND status = 'a'`. If 0 rows changed, someone else already moved it — stop, don't overwrite.
- **Inventory / limited seats:** decrement atomically with a `WHERE stock > 0` guard, or reserve with an expiring hold row.
- **Keep transactions short.** Never make a network call (Stripe, email, LLM) while holding a DB transaction or lock.

---

## 3. Failure & network rules

- **Every external call gets a timeout.** No default infinite waits. 5–15s for APIs, shorter for internal.
- **Retry only idempotent operations**, with exponential backoff + jitter, and a max attempt count. Pass an idempotency key so retries are safe.
- **The "unknown outcome" problem:** a request timed out — did it succeed? Never assume. Either (a) retry with the same idempotency key, or (b) query the provider for the real state, or (c) wait for the webhook. Never "just try again" without a key.
- **Partial failure:** if step 2 of 3 fails, the system must end in a known state. Use one of:
  - a single DB transaction (when all steps are local),
  - the **outbox pattern** (write the intent + the event in one transaction; a worker sends it),
  - a status column that a reconciliation job can resume from.
- **Client disconnects mid-request** (closed tab, lost wifi): the server keeps running. Design so the server finishes correctly and the client can **re-fetch status** when it comes back. Never make correctness depend on the client receiving a response.
- **Server crashes mid-request:** anything not committed is gone. Anything committed must be recoverable by a background job (look for rows stuck in intermediate states).
- **Circuit-break flaky dependencies.** If a provider is down, fail fast with a clear message instead of piling up hung requests.

---

## 4. Data & schema rules

- Money is **integer minor units** (cents) + a currency code. Never floats.
- Timestamps are `timestamptz`, stored in UTC. Convert at the edge.
- IDs: UUID/ULID for public-facing; never expose sequential IDs that leak volume.
- Every table: `created_at`, `updated_at`. Important tables: `status` + an append-only events/audit table.
- **Soft-delete** user-facing data; hard-delete only for legal requests.
- Migrations are **backward compatible**: add column → deploy code → backfill → enforce → remove old. Never rename/drop in the same deploy that stops using it.
- Add the index before the query goes to prod, not after the incident.

---

## 5. API rules

- Validate all input at the boundary with a schema (zod, pydantic). Reject unknown fields on sensitive endpoints.
- Authenticate, then **authorize on the resource**: "is this user allowed to touch *this* order?" — not just "is this user logged in?"
- Mutating endpoints accept an `Idempotency-Key` header; store `(key, user_id) → response` for 24h and replay it.
- Return stable error codes (`card_declined`, `already_paid`), not just messages.
- Rate-limit auth, payment, signup, and anything that costs you money (LLM calls, SMS, email).
- Paginate every list endpoint. Cursor-based for anything that grows.

---

## 6. Frontend rules

- Disable the submit button on click **and** send an idempotency key — the button alone is not protection (double taps, refresh, back button, two tabs).
- Show explicit states: idle → submitting → success / failed / **unknown (we're checking)**. Never leave the user guessing whether they were charged.
- After a payment redirect, the success page **asks the server** for status. It never marks anything paid itself.
- Optimistic UI only for cheap, reversible actions. Never for money.
- Handle the back button, refresh, and deep-linking into mid-flow pages.

---

## 7. Background jobs & webhooks

- Every job is idempotent and safe to run twice or concurrently.
- Store job state in the DB; claim work with `UPDATE ... SET locked_by = $me WHERE status = 'pending' ... FOR UPDATE SKIP LOCKED`.
- Jobs that die mid-way get picked up again (lease timeout), so they must resume, not restart from scratch blindly.
- Webhooks: verify signature → dedupe by event ID → store raw → return 2xx fast → process async. Events arrive **late, duplicated, and out of order**. Details in `references/payments.md`.
- Have a scheduled **reconciliation job** for anything that depends on an external system.

---

## 8. Security baseline

- Secrets in env/secret manager, never in code, logs, or the client bundle. Separate test/live keys per environment.
- Parameterized queries only. Escape output. CSP headers.
- Hash passwords (argon2/bcrypt) or, better, use a managed auth provider.
- Never log card data, full tokens, passwords, or full PII.
- Least privilege for DB users, API keys, and cloud roles.
- Dependency updates + `npm audit` / Dependabot on.

---

## 9. Observability

- Structured logs (JSON) with a `request_id` threaded through every call, job, and webhook.
- Log every state transition of important entities (order, payment, subscription) with before/after.
- Alerts on: error rate spikes, webhook failures, payments stuck in `pending` > N minutes, reconciliation mismatches, job queue backlog.
- Error tracking (Sentry or similar) in both frontend and backend.
- A health endpoint that checks the DB, not just "process is alive".

---

## 10. Shipping workflow

1. **Write the failure list first.** Before coding, list: what if it runs twice? concurrently? the network drops after the provider succeeds? the user refreshes? the webhook comes first? Design answers into the schema.
2. **Schema + constraints first**, then the server logic, then the UI.
3. **Test the unhappy paths**, not just the happy one: duplicate submit, concurrent submit (fire 20 parallel requests), timeout, provider error, webhook replay, out-of-order webhook.
4. **Feature flag** risky changes. Roll out to a % or internal users first.
5. **Have a rollback plan** before deploying. Know what happens to in-flight data if you roll back.
6. Run `references/preship-checklist.md`.
7. After deploy, watch logs and dashboards for 15–30 minutes. Don't ship and sleep.

---

## 11. Code style

- Small functions, obvious names, no premature abstraction. Duplicate twice, abstract on the third.
- Explicit over magic. A reader should see where money moves.
- Comments explain **why**, not what.
- Types at boundaries everywhere; `any` is a TODO.
- One way to do each thing in a codebase.
- Every bug fix ships with a test that would have caught it.
