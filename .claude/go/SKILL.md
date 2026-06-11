---
name: wms-go-development
description: Engineering standards for writing Go in the multi-tenant WMS SaaS backend (modular monolith on AWS Fargate, Aurora PostgreSQL, Redis). Use this skill whenever writing, reviewing, or refactoring ANY Go code for this project — handlers, services, repositories, SQL/migrations, concurrency and locking, event publishing, or tests. Consult it even for small changes, because this codebase has non-obvious correctness rules (tenant isolation via Postgres RLS, an append-only inventory ledger, oversell prevention, and a transactional outbox) that are easy to break silently. If a change touches tenants, inventory movements, stock, transactions, or events, this skill applies.
---

# WMS Go Development

Standards for the WMS SaaS backend. The goal is a **modular monolith** that stays clean enough to extract hot modules (e.g. the inventory core) into separate services later, without rewriting the system.


## Plan-first workflow (do this before writing code, every time)

This backend is where the most dangerous failure modes live: a leaked tenant query, a double-applied movement, an oversell. These are expensive to detect and worse to unwind in production. A short plan up front is the cheapest insurance:

1. **Restate the task** and identify which module(s) it touches and whether it reads or writes tenant/inventory data.
2. **Ask before assuming** on anything material — the exact workflow semantics, concurrency expectations, idempotency, what happens on partial failure. One good question beats a wrong implementation.
3. **Inspect existing code first** and match the established patterns (`WithTenantTx`, outbox, repo interfaces). Don't introduce a second way to do something that already has one.
4. **Write a short plan** before coding: files/modules to touch, the transaction boundaries, the events emitted, locking strategy, and the failure modes. Get sign-off for anything beyond a trivial change.
5. **Implement, then verify** against the checklist at the end of this skill — including the tenant-isolation and concurrency tests.

If the plan reveals a schema or infra change is also needed, coordinate via the `wms-database-development` / `wms-infra-development` skills rather than working around it.

## Architecture context

- Single Go binary (`/cmd/api`) deployed on **ECS Fargate (arm64 / Graviton)**.
- **Aurora PostgreSQL Serverless v2** is the system of record; behind **RDS Proxy**.
- **Redis/Valkey (ElastiCache)** for hot reservations and distributed locks.
- Multi-tenant **Pool** model: all standard tenants share schema, isolated by **Row-Level Security (RLS)** keyed on `tenant_id`. Enterprise tenants may be **siloed** to a dedicated DB later — code must never hardcode a DB endpoint.
- Supports B2C / B2B / 3PL as **configuration** over one core, not three systems. 3PL means one tenant owns many `owner_id`s (inventory owners). Standard warehouses have one owner == the tenant.

## Golden rules (do not violate)

1. **Every tenant-scoped query runs inside a tenant transaction.** Never query tenant data without first setting the tenant on the transaction (see Data Access). Fail closed: if there is no tenant in context, return an error — do not run the query.
2. **The movement ledger is append-only.** `INSERT` movements only. Never `UPDATE` or `DELETE` a movement row. Corrections are new compensating movements. This preserves the audit trail and lets balances be derived/snapshotted.
3. **The database is the final guard against oversell**, not Redis. A Redis lock can expire or fail; correctness must be enforced by the DB transaction (`SELECT ... FOR UPDATE` + check, or a constraint).
4. **Write the domain event in the same transaction as the data change** (transactional outbox). Never publish to the event bus directly from request handling.
5. **Modules talk through interfaces and the event bus only.** No module imports another module's internal packages.
6. **Propagate `context.Context` everywhere** (cancellation, deadlines, tenant). Never store request-scoped values in package globals.

## Project structure & module boundaries

```
/cmd/api/main.go
/internal/
  platform/        # shared infra ONLY (no business logic)
    db/            # pgx pool + tenant tx manager
    eventbus/      # EventBus interface (in-process now → EventBridge later)
    tenant/        # tenant_id <-> context.Context helpers
    auth/          # JWT verification, builds tenant context
  modules/
    inventory/     # ★ ledger, snapshot, reservation  (the hot core)
    inbound/       # receiving, QC, putaway
    outbound/      # pick, pack, ship
    returns/
    config/        # per-tenant workflow config engine
    task/          # operator task distribution + real-time
  controlplane/
    tenantreg/ routing/ billing/ provisioning/
/migrations/       # goose
```

Each module exposes a small public interface (e.g. `inventory.Service`) and keeps `domain/`, `service/`, `repo/`, `handler/` internal. Other modules depend on the **interface**, never the concrete type or its repo. Enforce this in CI with an import-boundary linter (e.g. `go-arch-lint`); a monolith without enforced boundaries rots into a big ball of mud regardless of language.

## Library stack (use these; do not add alternatives without discussion)

- Router: `chi`
- Postgres driver: `jackc/pgx/v5` (pool: `pgxpool`)
- Queries: **`sqlc`** (generate type-safe Go from hand-written SQL). Do **not** use a heavy ORM — this codebase needs precise control over transactions, RLS, locking, and partitioning.
- Migrations: `goose`
- Redis: `redis/go-redis/v9`; distributed locks: `go-redsync/redsync/v4`
- Logging: stdlib `log/slog` (structured; always include `tenant_id`)
- AWS: `aws-sdk-go-v2`
- WebSocket (real-time push): `coder/websocket`
- Tests: `testcontainers-go` (real Postgres)

## Data access & transactions (RLS)

RLS isolates tenants at the database layer so a coding mistake cannot leak another tenant's rows. It only works if the transaction declares its tenant **before** any query — and it must use a **transaction-local** setting so pooled connections (RDS Proxy reuses connections) never carry one tenant's id into another tenant's request.

**Use `set_config(..., true)` — the `true` makes it transaction-local. Never use a plain `SET` (session-level), which leaks across pooled connections.** Note `SET LOCAL x = $1` cannot be parameterized, so `set_config` is the correct form.

```go
// platform/db: run fn in a tenant-scoped, serializable transaction.
func (d *DB) WithTenantTx(ctx context.Context, fn func(tx pgx.Tx) error) error {
    tid, ok := tenant.FromContext(ctx)
    if !ok || tid == "" {
        return ErrNoTenant // fail closed
    }
    tx, err := d.pool.BeginTx(ctx, pgx.TxOptions{IsoLevel: pgx.Serializable})
    if err != nil {
        return err
    }
    defer tx.Rollback(ctx) // no-op after a successful Commit

    // transaction-local: auto-resets at COMMIT/ROLLBACK
    if _, err := tx.Exec(ctx,
        "SELECT set_config('app.current_tenant_id', $1, true)", tid); err != nil {
        return err
    }
    if err := fn(tx); err != nil {
        return err
    }
    return tx.Commit(ctx)
}
```

Every RLS policy filters on `current_setting('app.current_tenant_id')`. Repository methods accept a `pgx.Tx` (or a querier interface) — they never open their own pool connection.

## The inventory ledger

Stock is derived from an immutable log of movements, not a mutable balance column. Read state from the latest **snapshot** plus movements since the snapshot, so queries don't scan all history and old movements can be archived (Parquet on S3) without affecting current balances.

Append the movement and its event atomically:

```go
err := d.WithTenantTx(ctx, func(tx pgx.Tx) error {
    if err := q.InsertMovement(ctx, tx, mv); err != nil { // INSERT only
        return err
    }
    return q.InsertOutbox(ctx, tx, evt) // same tx → ledger and events never diverge
})
```

A separate relay goroutine reads unsent outbox rows, publishes to the EventBus, then marks them sent. Never publish events outside the transaction.

## Concurrency & oversell prevention (layered)

Fast path in Redis, hard guarantee in the DB:

```go
mutex := rs.NewMutex("lock:bin:"+binID, redsync.WithExpiry(5*time.Second))
if err := mutex.LockContext(ctx); err != nil {
    return ErrBusy
}
defer mutex.UnlockContext(ctx)

return d.WithTenantTx(ctx, func(tx pgx.Tx) error {
    // SELECT ... FOR UPDATE the stock row, verify available >= qty,
    // then append the reservation/pick movement. The DB is the source of truth.
})
```

Never trust the Redis lock alone. If the DB check passes only because Redis held a lock, an expired lock during a slow request would let two pickers reserve the same unit.

## Idempotency

Scanner/mobile clients are offline-tolerant and replay queued actions on reconnect. Every state-changing endpoint accepts an `Idempotency-Key`. Store processed keys (per tenant); on a repeat, return the original result instead of applying the action twice. Movement inserts must be idempotent on this key.

## Eventing

Define `EventBus` as an interface in `platform/eventbus`. In-process implementation for the monolith; an EventBridge implementation later. Use SQS FIFO for ordered work where movement order matters (e.g. per-SKU). This abstraction is what lets a module be extracted into its own service by swapping the implementation.

## Testing

- Use `testcontainers-go` to run a real Postgres (RLS and `FOR UPDATE` behavior cannot be tested against mocks).
- **Mandatory: a tenant-isolation test** — seed data for tenant A and tenant B, run queries as B, assert zero rows of A are ever visible. This test guards the most serious failure mode of a multi-tenant SaaS. Add one for every new tenant-scoped table.
- Concurrency tests: spawn parallel goroutines reserving the same stock; assert no oversell.

## Build & deploy

- Cross-compile for `linux/arm64`; ship a `distroless` or `scratch` image (tiny, fast cold start, ~20% cheaper on Fargate Graviton).
- Multi-stage Docker build; no build tools in the final image.
- Config from environment / Parameter Store; secrets from Secrets Manager — never hardcode.

## Before you commit — checklist

- [ ] All tenant data access goes through `WithTenantTx` (no raw pool queries).
- [ ] No `UPDATE`/`DELETE` on movement rows.
- [ ] Data change and its event share one transaction (outbox).
- [ ] Oversell is prevented by the DB, not only Redis.
- [ ] No cross-module internal imports; communication via interface/EventBus.
- [ ] `context.Context` threaded through; `tenant_id` in logs.
- [ ] New tenant-scoped table has an RLS policy and an isolation test.
- [ ] State-changing endpoints honor `Idempotency-Key`.


---

## v2.1 schema alignment — Odoo-style model (authoritative)

The inventory engine follows the Odoo stock model in `docs/WMS-Domain-Model.md`. Where this skill above says "movement"/"the ledger", the table is **`stock_move`** (merged plan+actual; `done` rows immutable). On-hand balance is **`stock_quant`** (`quantity` + `reserved_quantity`); there is no separate reservation table. Tracking via `lot`/`serial`/`label`; handling units via `stock_package`.

Stock moves between locations using **virtual locations** (`supplier`/`customer`/`inventory`) so every change is double-entry: receipt = `Supplier → Stock`, delivery = `Stock → Customer`, adjust = `Stock ↔ Inventory`. A move has `location_id` (source) and `location_dest_id` (destination); booking it decrements the source quant and increments the destination quant in the same tenant transaction.

**Invariants the Go data-access layer must uphold:**

- **R1** Reserve at availability check: `draft → assigned` bumps `stock_quant.reserved_quantity` (under `FOR UPDATE`), before the operator picks. `assigned → done` books `qty_done`.
- **R2** Enforce `available = quantity − reserved_quantity ≥ 0` ONLY for `internal` locations; virtual locations may go negative.
- **R3** Keep `reserved_quantity` equal to the sum of `assigned` moves — one `reserve()/release()` code path mutates move and quant together; never one without the other.
- **R4** Quantities are integers in base UoM (no fractional). Serial SKUs are 1-per-row.
- **R5** `done` moves are append-only; corrections are new compensating moves.

Update the "Before you commit — checklist" accordingly: tenant tx + `stock_quant` `FOR UPDATE` on every reservation/pick; reserved/quant mutated atomically; non-negativity checked only for internal locations.

---

## Project standards (as built — Sprint 1)

These reflect the actual repo. Follow them so new work matches what exists.

- **Location:** the backend lives in `backend/`; module is `github.com/wms-saas/wms`. Run all `go` / `make` commands from `backend/` (or use the root `Makefile`).
- **Layout:** `cmd/api` (entrypoint) · `internal/platform/{db,tenant,httpx}` (shared infra, NO business logic) · `internal/modules/<name>` (each module keeps domain + service + handler together) · `migrations/` (goose).
- **Pinned stack** (no alternatives without discussion): Echo (router), `jackc/pgx/v5` + `pgxpool`, stdlib `log/slog` (JSON), goose (migrations), `testcontainers-go` (tests). Serializable tenant transactions.
- **Tenant context:** `httpx.TenantMiddleware` puts the tenant on the request context (Sprint 1: `X-Tenant-Id` header stub; Sprint 2: Cognito JWT claim). Every tenant-scoped handler runs through `db.WithTenantTx` — never query the pool directly.
- **RLS insert pattern:** tenant-scoped tables set `tenant_id DEFAULT current_setting('app.current_tenant_id')::uuid`, so handlers/queries **never pass `tenant_id`** and the policy's `WITH CHECK` still passes. Reference: `migrations/00001_init_tenancy_rls.sql`.
- **Config:** from env via `internal/config`; hardcoded values are local defaults only.
- **Commands:** `make setup` (tidy), `make db-up`, `make migrate`, `make test`, `make run`. Commit `go.sum` (run `go mod tidy` first).
- **CI:** the `backend` job runs lint → build → test; the tenant-isolation test must pass.

---

## Dependency governance

Every dependency is a long-term liability (security, maintenance, breakage, license, lock-in). The default answer to "should we add this library?" is **no** until it clears the checklist in `docs/Dependency-Governance.md`.

- Stay on the **approved stack**. Anything new requires the add-a-dependency checklist **and** PR review.
- Prefer stdlib / an existing dep over a new one. **One library per concern** — no duplicates.
- **Pin exact versions; commit `go.sum`.** Never `go get ...@latest` into committed config.
- Permissive licenses only (MIT/Apache-2.0/BSD/ISC); no GPL/AGPL/SSPL without sign-off.
- **Enforced in CI:** `depguard` allow-list in `.golangci.yml` (an unlisted import fails the build — adding it to the list IS the approval), plus `govulncheck` and `go mod verify`.
