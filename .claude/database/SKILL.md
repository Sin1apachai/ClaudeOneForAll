---
name: database-development
description: Best-practice standards for relational databases (primarily PostgreSQL) — schema design, migrations, indexing, transactions, multi-tenant Row-Level Security, and data-layer testing. Use this skill whenever designing or changing schema, writing migrations, tuning queries, or implementing tenant isolation. It encodes general, reusable database practices; the project's concrete schema lives in its docs.
---

# Database Development

Reusable standards for relational data layers. The database is the last line of defense for correctness — design it to make wrong states impossible.

## Plan-first workflow

1. Restate the change and the tables/scope affected. Is it tenant-scoped? Online (zero-downtime)?
2. Ask before assuming on volume/growth, read/write patterns, and retention.
3. Inspect the existing schema and migrations; match naming and patterns.
4. Write a short migration plan (DDL, indexes, policies, expand/contract steps, rollback, locking risk); get sign-off for non-trivial changes.
5. Apply, then verify (isolation test + `EXPLAIN`).

## Schema design

- Clear, consistent names; appropriate types; `NOT NULL` + constraints to encode rules.
- Audit columns (`created_at`, `updated_at`, `created_by`) everywhere; `is_active`/status for master data instead of hard deletes.
- Enforce invariants with constraints (unique, check, foreign keys, exclusion) rather than trusting app code.

## Migrations

- Use a migration tool (e.g. goose); migrations are forward-only, numbered, one logical change per file, and reversible.
- **Online-safe (expand/contract):** add nullable/new first, backfill in bounded batches, switch reads, then drop old — never a destructive change that locks a hot table.
- `CREATE INDEX CONCURRENTLY` for big tables. Commit migrations to version control; never edit an applied/committed migration — add a new one.

## Indexing & performance

- Index for actual query patterns; lead composite indexes with the most selective / scoping column (e.g. `tenant_id`).
- `EXPLAIN (ANALYZE, BUFFERS)` new queries on representative data; a seq scan on a large table is a red flag.
- Keyset pagination over `OFFSET` for large lists; push aggregation into SQL (avoid N+1).

## Multi-tenant isolation (Row-Level Security)

When one database serves many tenants:

- Every tenant-scoped table has `tenant_id` + an RLS policy (`ENABLE` and `FORCE`), filtering on a **transaction-local** setting (`current_setting('app.current_tenant_id')`), set via `set_config(..., true)` — never a session-level `SET` (it leaks across pooled connections).
- **Critical:** the application must connect as a **non-superuser, non-`BYPASSRLS`** role. Superusers ignore RLS even with `FORCE`. Migrations run as the admin/owner; the app runs as a least-privilege role.
- Consider a `tenant_id` column DEFAULT of the current setting so inserts can't forget it and `WITH CHECK` still passes.

## Transactions & concurrency

- Wrap related writes in a transaction; pick the right isolation level. Enforce hard invariants (no oversell, uniqueness) with locks (`SELECT ... FOR UPDATE`) or constraints, not app logic alone.
- Append-only/event tables: insert only; corrections are compensating rows.

## Testing

- Test against real Postgres (RLS, locks, triggers can't be mocked).
- **Mandatory:** a tenant-isolation test for every tenant-scoped table — and run it **as the non-superuser app role**, or it passes falsely.

## Before you apply — checklist

- [ ] Plan + rollback written; signed off for non-trivial changes.
- [ ] New tenant-scoped table: `tenant_id`, RLS (USING + WITH CHECK), and an isolation test (as the app role).
- [ ] RLS uses a transaction-local setting; app connects as a non-superuser role.
- [ ] Invariants enforced by constraints/locks; append-only tables aren't UPDATE/DELETEd.
- [ ] Indexes match query patterns; `EXPLAIN` shows no surprise seq scan.
- [ ] Migration is online-safe (expand/contract, CONCURRENTLY, batched) and reversible.
