---
name: go-backend-development
description: Best-practice standards for building backend services in Go — project layout, idiomatic code, HTTP handlers, database access and transactions, concurrency, configuration, observability, and testing. Use this skill whenever writing, reviewing, or refactoring Go backend code (handlers, services, repositories, SQL/migrations, concurrency, tests). It encodes general, reusable Go practices; project-specific facts (chosen libraries, domain rules) live in the repo's CLAUDE.md.
---

# Go Backend Development

Reusable standards for Go services. Favor simple, explicit, well-tested code over cleverness.

## Plan-first workflow

1. Restate the task and the module it touches; note whether it reads or writes shared/sensitive data.
2. Ask before assuming on unclear semantics, concurrency, idempotency, and failure handling.
3. Inspect existing patterns and match them — don't introduce a second way to do something.
4. For non-trivial work, write a short plan (files, transaction boundaries, errors, edge cases) and get sign-off.
5. Implement, then verify against the checklist.

## Project layout

```
/cmd/<app>            # entrypoints (thin)
/internal
  /platform           # shared infra ONLY (db, config, http, auth) — no business logic
  /modules/<name>     # a module owns its domain + service + repo + handler
/migrations           # SQL migrations
```

Modules talk through small interfaces (and an event bus if used), never by importing another module's internals. Enforce boundaries in CI (e.g. an import-boundary linter) — a monolith without enforced boundaries rots.

## Idiomatic Go

- Propagate `context.Context` as the first arg; honor cancellation/deadlines. Never store request-scoped values in package globals.
- Wrap errors with context (`fmt.Errorf("...: %w", err)`); handle every error (`errcheck`).
- Accept interfaces, return structs; keep interfaces small and defined by the consumer.
- Prefer the standard library; add a dependency only via the project's dependency governance.
- `gofmt` + `go vet` + `staticcheck` clean; CI enforces.

## HTTP & middleware

- Pick one router and stick to it. Keep handlers thin: parse/validate → call service → encode response.
- Cross-cutting concerns (recover, request-id, auth) are middleware. Prefer small first-party middleware over pulling large packages with unwanted transitive deps.
- Validate and bound all input; return clear status codes; don't leak internals in error messages.

## Database & transactions

- Use `pgx` (+ `sqlc` for type-safe queries). Avoid heavy ORMs when you need precise control over transactions/locking.
- Parameterized queries only — never build SQL by string concatenation.
- Wrap related writes in a transaction. For multi-tenant Postgres with RLS: set the tenant on a **transaction-local** setting (`set_config(..., true)`) and connect as a **non-superuser** role (superusers bypass RLS). Fail closed if no tenant in context.
- Enforce hard invariants (no oversell, uniqueness) in the DB (constraints / `SELECT ... FOR UPDATE`), not only in app code.
- Append-only/event tables: insert only; corrections are new compensating rows.

## Concurrency

- Use goroutines deliberately; always have a way to stop them (context). Avoid leaks.
- Protect shared state; prefer channels or `sync` primitives; run tests with `-race`.

## Config, secrets, observability

- Config from environment; secrets from a secret store, never hardcoded or logged.
- Structured logging (`log/slog`); include correlation/tenant ids; never log secrets or PII.
- Idempotency keys for state-changing endpoints that clients may retry.

## Testing & build

- Table-driven unit tests; real Postgres via `testcontainers-go` for anything touching SQL/RLS/locks.
- Multi-stage Docker → minimal/distroless image; build for the target arch; no build tools in the final image.

## Before you commit — checklist

- [ ] Plan written + signed off for non-trivial work.
- [ ] `context` threaded; errors wrapped and handled; `-race` clean.
- [ ] Parameterized SQL; related writes in one transaction; invariants enforced in DB.
- [ ] (Multi-tenant) tenant set transaction-locally; app connects as a non-superuser role; isolation tested.
- [ ] No cross-module internal imports; new deps cleared by governance.
- [ ] Structured logs without secrets/PII; lint/vet/staticcheck clean.
