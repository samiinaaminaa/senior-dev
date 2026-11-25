# Pre-ship checklist

Go through every section that applies. Anything unchecked is either fixed or written down as a known risk.

## Correctness under failure
- [ ] Every mutating endpoint is idempotent (idempotency key or natural unique key)
- [ ] Every "only one of X" rule is a DB unique constraint, not an `if`
- [ ] No read-modify-write on shared values; atomic SQL updates instead
- [ ] State changes use `UPDATE ... WHERE status = <expected>` and check row count
- [ ] No network calls inside DB transactions
- [ ] Every external call has a timeout and bounded retries
- [ ] Timeouts are treated as "unknown", not "failed"
- [ ] Rows stuck in intermediate states are picked up by a job

## Payments (if applicable — see payments.md)
- [ ] Order created in DB before calling the gateway
- [ ] Gateway idempotency key derived from order ID
- [ ] Only webhook / server-side verification marks orders paid
- [ ] Webhook signature verified on raw body; events deduped by ID
- [ ] Out-of-order and duplicate webhooks can't downgrade or double-fulfill
- [ ] Amount and currency from gateway compared to stored order
- [ ] Price computed server-side; client can't change it
- [ ] Reconciliation job running and alerting
- [ ] Real live-mode transaction tested and refunded
- [ ] Test and live keys/webhook secrets separated per environment

## Security
- [ ] Authorization checked on the specific resource, not just login
- [ ] All input validated with a schema
- [ ] No secrets in code, logs, or client bundle
- [ ] Rate limits on auth, payment, signup, and paid-API endpoints
- [ ] No PII, tokens, or card data in logs

## Data
- [ ] Migrations backward compatible with the currently running code
- [ ] Indexes exist for new queries
- [ ] Money in integer minor units; timestamps in UTC `timestamptz`
- [ ] Backup exists and restore has been tried at least once

## UX
- [ ] Buttons disable on submit; loading, error, and "processing" states shown
- [ ] Refresh, back button, and two tabs behave correctly
- [ ] Error messages are human and actionable; no stack traces shown

## Observability
- [ ] Structured logs with request IDs
- [ ] Error tracking in frontend and backend
- [ ] Alerts for error spikes, stuck states, webhook failures, payments silently stopping

## Release
- [ ] Unhappy-path tests pass (duplicates, concurrency, timeouts)
- [ ] Feature flag or staged rollout for risky changes
- [ ] Rollback plan written, including what happens to in-flight data
- [ ] Someone watches dashboards for 15–30 min after deploy
