---
name: wms-database-development
description: Engineering standards for the WMS SaaS data layer on Aurora PostgreSQL (multi-tenant Pool model with Row-Level Security, plus optional Enterprise Silo). Use this skill whenever designing or changing ANYTHING in the database — schema, tables, columns, indexes, RLS policies, migrations, partitioning, constraints, or query/SQL performance. Consult it even for a one-column change, because this schema has non-obvious correctness rules (RLS-enforced tenant isolation, an append-only inventory ledger, DB-enforced oversell prevention, expand/contract migrations on a live multi-tenant DB) that are easy to break silently and can leak data across tenants. If a change touches tenant tables, inventory/stock, migrations, or indexes, this skill applies.
---

# WMS Database Development

Standards for the WMS data layer. The database is the **system of record and the final guard** for the most dangerous failure modes of a multi-tenant SaaS: cross-tenant data leakage and overselling stock. The Go data-access rules live in the `wms-go-development` skill; this skill governs schema, RLS, migrations, indexing, and SQL.

## Plan-first workflow (do this before any schema/SQL change, every time)

A schema change on a live, shared, multi-tenant database is high-stakes: a missing RLS policy leaks every tenant's rows, a blocking migration locks the floor out of the system, and a bad index lets one big tenant slow everyone down. Two minutes of planning saves a production incident:

1. **Restate the change** and name the tables/tenants affected. Is the table tenant-scoped? (Almost everything is.)
2. **Ask before assuming** on anything material — expected row volume and growth, read/write patterns, whether it must be online (zero-downtime), retention/archival needs. These determine indexing and partitioning, and they're cheap to ask now.
3. **Inspect the existing schema and migrations first** so the change matches established naming, types, and the RLS pattern. Never invent a second pattern for the same problem.
4. **Write a short migration plan**: the exact DDL, the RLS policy, the indexes, the expand/contract steps if online, the rollback, and the data-volume/locking risk. Share it and get sign-off before applying anything beyond a trivial additive change.
5. **Apply, then verify**: run the isolation test and check query plans (`EXPLAIN`) against the checklist at the end.

If the change implies application-code changes (new RLS context, new query), flag them in the plan so backend and DB move together.

## Architecture context

- **Aurora PostgreSQL Serverless v2**, accessed through **RDS Proxy** (connection pooling for serverless/Fargate). Capacity scales on ACUs — schema and queries must be efficient because waste shows up directly on the bill.
- **Pool model**: standard tenants share every table, isolated by **Row-Level Security** keyed on `tenant_id`. **Enterprise tenants may be siloed** to a dedicated database later — so nothing may hardcode a tenant→endpoint mapping; routing is the app/control-plane's job.
- One core schema supports B2C / B2B / 3PL as configuration. 3PL means a tenant owns many inventory `owner_id`s; model ownership explicitly.

## Golden rules (do not violate)

1. **Every tenant-scoped table has `tenant_id NOT NULL`, an RLS policy, and a tenant-isolation test.** No exceptions. A new table without a policy is a data breach waiting to happen.
2. **RLS filters on a transaction-local setting**, never a session setting. Pooled connections (RDS Proxy) are reused across tenants; a session-level `SET` would leak one tenant's id into another's request. Use `current_setting('app.current_tenant_id')` populated via `set_config(..., true)`.
3. **The inventory movement ledger is append-only.** `INSERT` only — never `UPDATE`/`DELETE` a movement. Corrections are compensating movements. This is the audit trail and the basis for derived balances.
4. **The database enforces no-oversell**, not the application cache. Use `SELECT ... FOR UPDATE` + check, or a `CHECK`/exclusion constraint. Redis is an optimization, not a guarantee.
5. **Migrations are forward-only and online-safe.** Use expand/contract; never a destructive change that locks a hot table or breaks the currently-deployed app.
6. **Index for the Pool.** Every index on a tenant-scoped table leads with `tenant_id`. A query that scans across tenants is both a performance bug and a noisy-neighbor risk.

## RLS pattern (the canonical form)

Enable RLS and force it (so even the table owner is subject to it), and write policies against the transaction-local tenant:

```sql
ALTER TABLE stock_move ENABLE ROW LEVEL SECURITY;
ALTER TABLE stock_move FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON stock_move
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid)
  WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

The `USING` clause filters reads; `WITH CHECK` prevents inserting/updating a row into another tenant. The application sets the tenant once per transaction (`SELECT set_config('app.current_tenant_id', $1, true)`); the DB does the rest. Add the matching policy in the **same migration** as the table — never in a later one.

## Inventory ledger & balances

Stock is **derived from an immutable log of movements**, not a mutable balance column, so history is auditable and corrections are non-destructive:

- `stock_move`: append-only, one row per stock change (receipt, putaway, pick, adjust, transfer), with `tenant_id`, `owner_id`, `sku_id`, `location_id`, signed `qty`, `reason`, and an `idempotency_key`.
- **Snapshots**: periodically materialize a balance per (tenant, owner, sku, location) so reads = latest snapshot + movements since. This keeps reads cheap and lets old movements be archived (Parquet on S3) without affecting current balances.
- **Partition** large append-only/event tables (movements, outbox, audit) by time (monthly range) so writes stay on a small hot partition and old data is cheap to detach/archive.

## Migrations (goose, expand/contract)

On a live multi-tenant DB you can't take downtime for a schema change, so split changes into compatible steps:

1. **Expand**: add the new nullable column/table/index (use `CREATE INDEX CONCURRENTLY` to avoid locking writes). Deploy code that writes both old and new.
2. **Backfill** in batches (bounded by `tenant_id`) to avoid long locks and ACU spikes.
3. **Contract**: once all code reads the new shape, drop the old column/constraint in a later migration.

Rules: one logical change per migration; always reversible or explicitly documented as irreversible with a rollback plan; never `ALTER` that rewrites or exclusively locks a hot table during business hours.

## Indexing & query performance

- Lead composite indexes with `tenant_id`, then the next most selective column the query filters on.
- `EXPLAIN (ANALYZE, BUFFERS)` any new query on representative data before shipping; a seq scan on a tenant table is a red flag.
- Prefer keyset/seek pagination over `OFFSET` for large lists (order queues, movement history).
- Watch for N+1 from the app; push aggregation into SQL.

## Cost & multi-tenant hygiene (Aurora Serverless v2)

- Inefficient queries raise ACU usage and the bill — treat slow queries as cost bugs, not just latency bugs.
- Keep the hot working set small (snapshots + partitioning) so Aurora can scale ACUs down when idle.
- No tenant→DB endpoint hardcoding anywhere; siloing an Enterprise tenant must be a config/routing change, not a schema fork.

## Before you apply — checklist

- [ ] Wrote a migration plan and got sign-off for anything beyond a trivial additive change.
- [ ] New tenant-scoped table has `tenant_id NOT NULL` + RLS policy (USING and WITH CHECK) in the same migration.
- [ ] A tenant-isolation test covers the new/changed table (seed tenant A & B; as B, assert zero A rows).
- [ ] RLS uses the transaction-local setting, never a session `SET`.
- [ ] No `UPDATE`/`DELETE` path was added to the movement ledger.
- [ ] Oversell is prevented by a DB lock/constraint, not only by application logic.
- [ ] Indexes lead with `tenant_id`; `EXPLAIN` shows no unexpected seq scan.
- [ ] Migration is online-safe (expand/contract, `CONCURRENTLY`, batched backfill) and has a rollback.
- [ ] Large append-only/event tables are partitioned by time.


---

## v2.1 schema alignment — Odoo-style model (authoritative)

The data layer follows the Odoo stock model documented in `docs/WMS-Domain-Model.md`. Use these names and rules; they supersede any older `inventory_movement`/`STOCK_BALANCE` references above.

**Tables:** `stock_location` (with `usage`: view/internal/supplier/customer/inventory/transit), `stock_picking` (+ `stock_picking_type`, `stock_picking_batch`), `stock_move` (merged plan+actual ledger — append-only, `done` rows immutable), `stock_quant` (on-hand balance: `quantity` + `reserved_quantity`), `stock_package` (LPN/pallet), `lot` / `serial` / `label` for tracking.

**Double-entry via virtual locations:** every movement is `location_id → location_dest_id`. Receipt = `Supplier → Stock`, delivery = `Stock → Customer`, adjustment = `Stock ↔ Inventory`. No special-case "in/out" columns.

**Rules to enforce in DB (triggers/constraints) + data-access layer:**

- **R1 Reserve at availability check** — move `draft → assigned` bumps `stock_quant.reserved_quantity` under `FOR UPDATE`, before physical pick. `available = quantity − reserved_quantity`.
- **R2 Non-negativity, internal only** — `available ≥ 0` enforced ONLY for quants whose location `usage = internal`. Virtual locations may go negative. Use a `FOR UPDATE` check in the booking tx + a `BEFORE INSERT/UPDATE` trigger on `stock_quant` that rejects `quantity < 0` when the location is internal (a plain `CHECK` can't see usage — denormalize `location_usage` onto the quant if you prefer a constraint).
- **R3 Reserved invariant** — `stock_quant.reserved_quantity` must equal `SUM(qty)` of `assigned` moves on that quant. Maintain both in one transaction via a single `reserve()/release()` path; add a reconciliation query as a test + periodic audit to catch ghost reservations.
- **R4 Integer quantities** — all stock quantities are whole numbers in base UoM; no fractional columns. Finer granularity = a smaller base UoM.
- **R5 Immutable done moves** — never `UPDATE`/`DELETE` a `done` `stock_move`; corrections are compensating moves.

Add these to the "Before you apply — checklist": (a) new internal-location writes guarded for non-negativity; (b) any reservation change touches move + quant atomically; (c) quantities are integer.

---

## Project standards (as built — Sprint 1)

These reflect the actual migration shape. Copy it for every new table.

- **Location/format:** migrations live in `backend/migrations/`, goose format (`-- +goose Up/Down`, `StatementBegin/End`), numbered `0000N_name.sql`, one logical change per file, reversible. Apply with `make migrate`.
- **Reference:** `00001_init_tenancy_rls.sql`. Every tenant-scoped table must replicate its shape:
  - `tenant_id uuid NOT NULL DEFAULT current_setting('app.current_tenant_id')::uuid`
  - `ENABLE` **and** `FORCE ROW LEVEL SECURITY`
  - `CREATE POLICY tenant_isolation ... USING (tenant_id = current_setting('app.current_tenant_id')::uuid) WITH CHECK (same)`
  - audit columns `created_at` / `updated_at`; `is_active` on master data.
- The `tenant_id` DEFAULT means inserts omit `tenant_id` and `WITH CHECK` still passes.
- **Mandatory:** add a tenant-isolation test (see the `product` template) for every new tenant-scoped table.
