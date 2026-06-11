---
name: wms-testing
description: Engineering standards for testing the WMS SaaS — unit tests, smoke tests, and automation tests (integration, contract, e2e, load). Use this skill whenever writing, reviewing, running, or debugging ANY test for this project: Go `_test.go` files, the smoke-test script, testcontainers integration tests, Playwright e2e, or CI test config. Consult it even for a small test, because this codebase has correctness rules that MUST be guarded by tests (tenant isolation via RLS, no-oversell, the reserved-quantity invariant, idempotency) — an untested tenant-scoped change is a data-leak risk. If a change touches tenant data, inventory/stock, or adds an endpoint, a test is required.
---

# WMS Testing

How we keep a multi-tenant inventory system correct. Pairs with `docs/Test-Strategy.md`. Three layers, named as the team uses them: **unit**, **smoke**, **automation**.

## Plan-first workflow (before writing tests)

1. **State what you're protecting** in one line, and which failure it prevents (cross-tenant leak? oversell? wrong state transition?).
2. **Pick the right layer.** Unit for pure logic; integration for anything touching SQL / RLS / locks; smoke for a running-system path; e2e for an operator flow. Testing at the wrong layer gives false confidence — RLS and locking break *only* against real Postgres, never against a mock.
3. **Reuse the existing template** (the tenant-isolation test, the smoke script) instead of inventing a new shape.
4. **For tenant-scoped or stock-mutating changes a test is not optional** — write it in the same PR.

## Philosophy

The two failure modes that can kill the business are **cross-tenant data leakage** and **overselling stock**. Tests exist first to guard these; everything else is secondary. Prefer real Postgres over mocks wherever correctness depends on the database.

## 1. Unit tests

Fast, in-memory, no DB or network. Test domain logic, UoM conversion, state-machine transitions, validation, and pure helpers.

- **Go:** table-driven, stdlib `testing`; file `<name>_test.go`; prefer black-box `package <pkg>_test`. Use `t.Parallel()` where safe; no sleeps; deterministic.
- **Frontend:** Vitest + React Testing Library for components/hooks; mock the `api.ts` client (don't hit a real backend).
- Run: `cd backend && go test ./...` · `cd frontend && npm test` (add Vitest when components land).

## 2. Smoke tests

A thin **black-box** check that a *running* system's critical paths work end-to-end. Lives in `scripts/smoke-test.sh`; run via `make smoke` against `make run-backend`.

- Asserts: health, **fail-closed auth** (401 with no tenant), create/list, and **tenant isolation** (A can't see B).
- Rules: idempotent; fresh random tenants per run; no leftover fixtures that matter; fail loudly with a clear message.
- Run after **every deploy** and before a demo.
- **Extend it** whenever you add a user-facing endpoint — one happy path plus the isolation/permission boundary.

## 3. Automation tests

The CI-gating, repeatable suite.

**Integration (testcontainers-go, real Postgres)** — RLS, `SELECT ... FOR UPDATE`, triggers, and the `tenant_id` DEFAULT cannot be mocked. Template: `backend/internal/modules/product/isolation_test.go`. Mandatory (gate the build):

- **Tenant isolation** — one per tenant-scoped table (copy the template): seed tenant A + B, act as B, assert zero of A's rows are visible.
- **No oversell** — concurrent reserve/pick on the same stock; `available` never goes negative on `internal` locations.
- **Reserved invariant** — `stock_quant.reserved_quantity == Σ qty of assigned moves`.
- **Idempotency** — replaying an action with the same key applies once.
- **Migration** — `up` then `down` runs clean on a fresh DB.

**Contract** — front↔back validated against the OpenAPI spec (when it lands).
**E2E (Playwright)** — critical operator flows: receive→putaway, pick→pack→ship, against a seeded test warehouse.
**Load (k6 / Locust)** — scan throughput and concurrent picking.

## Conventions

- **Real Postgres** for anything touching SQL/RLS/locks — never a mock DB.
- Clean up with `t.Cleanup`; tests never depend on order or shared mutable state.
- Deterministic data: fixed UUIDs/seeds where assertions need them; random tenants where you only need separation.
- One theme per test; names read clearly in CI output.
- Reuse the tenant-isolation template for every new tenant-scoped table — don't reinvent it.

## CI gates (`.github/workflows/ci.yml`)

- `backend` job: lint → build → `go test ./...` (incl. integration). Green required to merge to `main`.
- `frontend` job: install → build (+ `npm test` when present).
- Smoke runs as a post-deploy step, not in the unit CI job.

## Before you commit — checklist

- [ ] New tenant-scoped table has a tenant-isolation test (from the template).
- [ ] New endpoint has a unit/integration test + a smoke case (happy path + boundary).
- [ ] Stock-mutating logic has a concurrency/no-oversell test and upholds the reserved invariant.
- [ ] SQL/RLS/lock behavior is tested against real Postgres, not a mock.
- [ ] `make test` green locally; migrations up+down clean.
