# Testing

Use the runner the repo already has (Vitest, Jest, or similar) and the setup its existing
tests use for the database and the session.

## Cases for a new action

Each mutating action is covered for:

- Invalid input → failure response, no write.
- No session → failure, no write.
- Session without permission → failure, no write.
- Record from another tenant → failure, record unchanged.
- Related ID from another tenant → failure, no write.
- Nonexistent record → failure.
- Successful create and successful update → expected rows and response shape.
- Failure partway through the transaction → no partial writes remain.
- External effect fails (email, WhatsApp) → confirmed database state is intact.

Read actions are covered for tenant scope (another tenant's rows never appear) and, when
paginated, for `page`, `totalPages`, and `totalCount`.

## Weak existing tests

A test that only asserts it is defined, or asserts nothing about behavior, is not coverage.
Name such tests in the report.

## Reporting tests

The report separates tests that exist, tests that were run (with results), and tests that
were not run (with the reason).
