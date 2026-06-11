---
name: wms-security
description: Cross-cutting security standards for the WMS SaaS — secure coding, secrets, authn/authz, tenant isolation, input handling, dependencies, and the always-on threat checklist. Use this skill on EVERY change that touches code, data, dependencies, infrastructure, or configuration — not only on "security work". This is a multi-tenant SaaS holding customer data, so almost any change can introduce a tenant-data leak, an injection, a secret exposure, or a supply-chain risk. If you are writing or reviewing code, handling input, adding a dependency, touching auth/RLS, or changing infra, consult this skill first.
---

# WMS Security

Security is not a feature or a phase — it is a property every change must preserve. This is a multi-tenant SaaS holding customers' (and their customers') data; the cost of getting it wrong is a breach. Treat every change as a potential attack surface.

## Plan-first: ask "how could this be abused?"

Before writing anything that handles input, data, auth, secrets, or external calls, name the threat: what is the worst a malicious tenant, a buggy client, or an attacker could do here? Design against it, then write.

## The always-on checklist (every change)

- **Tenant isolation preserved** — queries go through `WithTenantTx`; a new tenant-scoped table has RLS + an isolation test. Never widen access.
- **Input validated and bounded** — never trust client input (type, size, range). **Parameterized SQL only** — never build queries by string concatenation.
- **Authorization server-side** — check the permission in the handler; never rely on the UI hiding things.
- **Secrets out of code & logs** — from Secrets Manager / env only; never log tokens, passwords, PII, or full tenant data.
- **Output encoded** — no reflected/stored XSS on the frontend; security headers set.
- **Errors don't leak internals** — generic messages to clients; detail to logs.
- **New dependency cleared** — `docs/Dependency-Governance.md` (vuln scan + license + allow-list).
- **Least privilege** — IAM, DB roles, and tokens scoped to the minimum.

## Per-layer essentials

- **Backend:** RLS fail-closed; pgx parameterized queries; idempotency keys; per-tenant rate limit; no secrets in code; logs without PII.
- **Frontend:** never ship secrets in the bundle; sanitize/encode output; CSP + security headers; the JWT/tenant claim is the source of truth, not client state.
- **Database:** `FORCE` RLS on every tenant table; least-privilege DB roles; encryption at rest (KMS); no PII in non-prod dumps.
- **Infra:** private subnets; least-privilege IAM (no `*`); TLS everywhere; WAF; secrets in Secrets Manager; encrypt + deletion-protect stateful resources.

## Keeping it current (automated, not memorized)

Security drifts as new CVEs and threats appear, so the project leans on continuous tooling rather than anyone's memory:

- **Local-first (do this while you dev):** run `make security` after changes; the committed git hooks (`.githooks/`, enabled by `make setup`) run fmt/vet on commit and `make security` on push. Fix what you find here — don't rely on the GitHub bots to catch it for you.
- **CI scanners** (`.github/workflows/security.yml`): CodeQL (SAST), gosec (Go), TruffleHog (secrets) — plus `govulncheck` / `npm audit` from the dependency CI. They run on every PR **and on a weekly schedule** against fresh advisory data.
- **Dependabot** (`.github/dependabot.yml`) opens PRs for vulnerable/outdated deps (gomod, npm, actions).
- **Living standard + threat model**: `docs/Security.md`, reviewed quarterly.
- **PR template** forces a security check on every change.

## When in doubt

Stop and flag it — a 5-minute question beats a breach. **Never disable a security control** (RLS, auth, a scanner) to "make it work"; fix the cause.
