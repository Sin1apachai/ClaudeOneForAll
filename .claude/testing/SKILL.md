---
name: testing
description: Best-practice standards for testing software — unit tests, smoke tests, and automation tests (integration, contract, e2e, load). Use this skill whenever writing, reviewing, running, or debugging tests. It encodes general, reusable testing practices; the project's specific critical invariants and mandatory tests live in its docs.
---

# Testing

Reusable testing standards. Tests exist first to protect the few failure modes that would seriously hurt the product — identify those and guard them above all.

## Plan-first workflow

1. State what behavior you're protecting and which failure it prevents.
2. Pick the right layer (below). Testing at the wrong layer gives false confidence — things that only break against a real database/network must be tested there, not against mocks.
3. Reuse existing test templates instead of inventing a new shape.
4. For changes to critical or shared logic, a test is not optional — write it in the same PR.

## The three layers

**Unit** — fast, in-memory, no DB/network. Test pure logic, transformations, validation, and state transitions. Table-driven; deterministic (no sleeps); run with `-race` (Go) where relevant. Frontend: component/hook tests with mocked I/O.

**Smoke** — a thin black-box check that a *running* system's critical paths work end-to-end (health, auth/fail-closed, a happy path, and any security boundary like multi-tenant isolation). Idempotent, uses fresh data per run, fails loudly. Run after every deploy. Extend it whenever you add a user-facing endpoint.

**Automation** — the CI-gating, repeatable suite:
- *Integration* against real dependencies (e.g. a real database via testcontainers) — the only place RLS, locking, and triggers can be verified. Run security-sensitive checks as the same low-privilege role the app uses, or they pass falsely.
- *Contract* — front↔back validated against the API spec.
- *E2E* — critical user flows, against seeded data.
- *Load* — throughput and concurrency on hot paths.

## Mandatory test patterns (adapt per project)

- **Isolation** — for any multi-tenant/scoped table, assert one tenant can never see another's rows.
- **Invariant under concurrency** — for any "must never go wrong" rule (no oversell, no double-apply), spawn concurrent operations and assert the invariant holds.
- **Idempotency** — replaying an action with the same key applies once.
- **Migrations** — `up` then `down` runs clean on a fresh DB.

## Conventions

- Real dependencies where correctness depends on them; never a mock DB for SQL/RLS/locks.
- Clean up with teardown hooks; no order dependence or shared mutable state.
- Deterministic data where assertions need it; random separation where you only need uniqueness.
- One theme per test; names that read clearly in CI output.

## CI gates

- Lint → build → unit + integration must pass to merge. Coverage threshold on core logic. Migration up/down clean. Smoke runs post-deploy.

## Before you commit — checklist

- [ ] New scoped table has an isolation test (run as the low-privilege role).
- [ ] New endpoint has a unit/integration test + a smoke case (happy path + boundary).
- [ ] Critical invariants have a concurrency test.
- [ ] SQL/lock/RLS behavior tested against real infrastructure, not mocks.
- [ ] Tests green locally; migrations up+down clean.
