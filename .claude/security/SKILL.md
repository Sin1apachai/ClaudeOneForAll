---
name: security
description: Cross-cutting security standards for application development — secure coding, secrets, authentication/authorization, multi-tenant isolation, input handling, dependencies, and an always-on threat checklist. Use this skill on EVERY change that touches code, data, dependencies, infrastructure, or configuration — not only on "security work". It encodes general, reusable security practices; the project's threat model and tooling live in its docs.
---

# Security

Security is a property every change must preserve, not a phase. Treat every change as a potential attack surface and ask "how could this be abused?" before writing.

## The always-on checklist (every change)

- **Isolation preserved** — for multi-tenant systems, access stays scoped (e.g. RLS via a transaction-local setting, app connecting as a non-superuser role). Never widen access.
- **Input validated and bounded** — never trust client input (type, size, range). **Parameterized queries only** — never string-built SQL.
- **Authorization server-side** — check permissions in the handler; never rely on the UI hiding things.
- **Secrets out of code & logs** — from a secret manager; never log tokens, passwords, or PII.
- **Output encoded** — prevent XSS; set security headers / CSP.
- **Errors don't leak internals** — generic messages to clients, detail to logs.
- **New dependency cleared** — necessary, maintained, permissive license, vuln-scanned, pinned (see the project's dependency governance).
- **Least privilege** — IAM, DB roles, and tokens scoped to the minimum.

## Per-layer essentials

- **Backend:** fail-closed auth; parameterized queries; idempotency keys; rate limiting; no secrets in code; logs without PII.
- **Frontend:** no secrets in the bundle; encode output; CSP + security headers; trust the auth token, not client state.
- **Database:** `FORCE` RLS on tenant tables + a non-superuser app role; least-privilege grants; encryption at rest.
- **Infra:** private subnets; least-privilege IAM (no `*`); TLS everywhere; secrets in a secret manager; encrypt + protect stateful resources.

## Keeping it current (automated, not memorized)

Security drifts as new CVEs appear, so lean on continuous tooling, not memory:

- **Local-first:** run the project's security scans (vuln + SAST + secret scan) while developing; git hooks gate commit/push. Fix what you find — don't rely on CI/bots to catch it later.
- **CI scanners** (SAST/CodeQL, secret scanning) run on every PR and on a schedule against fresh advisory data.
- **Automated dependency updates** (Dependabot/Renovate), reviewed not blind-merged.
- A living **threat model** reviewed on a cadence; a **PR template** that forces a security check.

## When in doubt

Stop and flag it — a 5-minute question beats a breach. **Never disable a security control** (RLS, auth, a scanner) to "make it work"; fix the cause.
