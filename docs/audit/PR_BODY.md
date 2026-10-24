## Summary

This PR completes the repository audit and hardens the GST and multi-branch inventory
paths without claiming unfinished business workflows are complete.

## Bugs fixed

- BUG-04: odd-paise intra-state GST split now conserves total tax exactly.
- BUG-05: transfer creation supports unique idempotency keys.
- BUG-06: stock batch and transfer money fields use `Numeric(12, 2)` and Decimal values.
- BUG-07: dispatched transfers can be cancelled with source-stock restoration.
- BUG-08/09/10/21: existing GST checksum, credit-note, signing fallback, and vehicle
  validation fixes were verified.

## Inventory and API changes

- Added transfer cancellation and stock-conservation regression coverage.
- Added failure-injection rollback coverage.
- Exposed `idempotency_key` through the transfer API.
- Added Alembic revision `7a2f6e1c4b90` for money-field and idempotency schema changes.

## GST changes

Updated regression expectations for the required CGST + SGST == total-tax invariant and
the resulting invoice round-off.

## Tests

- Full suite: 514 passed.
- Coverage: 84% for `app`.
- `compileall`: passed.
- Clean SQLite Alembic upgrade: passed through `7a2f6e1c4b90`.
- Full model registration/database initialization and API import smoke checks: passed.

## Documentation

README setup, API, desktop, test, coverage, and known-limitation commands were verified.
The audit report and progress log document the remaining scaffolded modules.

## Known limitations

Business setup, POS billing, customer Udhaar, reports/exports, authentication wiring, and
operational alerts remain incomplete and are explicitly marked as such in the audit.

## Conflict notes

Open upstream inventory PRs #4, #6, and #7 overlap with transfer validation/API areas.
This branch preserves existing route contracts and records the overlap; no upstream PR
was merged or closed.

Full audit: [`docs/audit/AUDIT_REPORT.md`](./AUDIT_REPORT.md)
