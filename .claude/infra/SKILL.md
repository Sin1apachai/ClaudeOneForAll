---
name: wms-infra-development
description: Engineering standards for the WMS SaaS infrastructure on AWS, defined as code with AWS CDK (TypeScript) — Fargate, Aurora Serverless v2, RDS Proxy, ElastiCache, SQS, Cognito, S3/CloudFront, WAF, networking, IAM, CI/CD, and observability. Use this skill whenever creating or changing ANY infrastructure — CDK stacks/constructs, IAM policies, networking, environment config, pipelines, alarms, or anything affecting cost or the Pool/Silo tenant topology. Consult it even for small changes, because this platform has non-obvious rules (bootstrap cost discipline, least-privilege IAM, per-tenant cost tagging, no click-ops, Pool-vs-Silo provisioning) that are easy to break and expensive or insecure to get wrong. If a change touches AWS resources, deployment, secrets, or cost, this skill applies.
---

# WMS Infra Development

Standards for the WMS SaaS platform on AWS. Two non-negotiables shape everything: **bootstrap cost discipline** (we pay close to usage, not peak) and **multi-tenant safety** (Pool by default, Silo for Enterprise, with per-tenant cost visibility). All infrastructure is **code** — there is no manual console change that survives.

## Plan-first workflow (do this before any infra change, every time)

Infrastructure changes are slow to reverse and easy to make expensive or insecure: an over-broad IAM policy is a security hole, a forgotten NAT Gateway or always-on instance quietly drains the runway, and a stateful resource deleted by a careless `cdk destroy` loses data. So plan before you touch a stack:

1. **Restate the change** and name the environment(s) and resources affected (dev / staging / prod). Production changes get extra scrutiny.
2. **Ask before assuming** on anything material — cost impact, whether it's tenant-scoped (Pool shared vs Silo per-tenant), data-loss risk, blast radius, expected scale. A two-line question is cheaper than a surprise bill or outage.
3. **Inspect existing stacks/constructs first** and reuse the established patterns (naming, tagging, env config). One way to do a thing.
4. **Write a short plan**: which stack, the resources added/changed, the estimated monthly cost delta, IAM/security impact, and how to roll back. Always **run `cdk diff` and read it** before deploying. Get sign-off for anything touching prod, IAM, networking, data stores, or cost.
5. **Deploy, then verify**: confirm the diff matched intent, tags are present, alarms fire, and nothing drifted. Never leave a console hot-fix undocumented in code.

If a change can't be expressed in CDK, that's a signal to stop and rethink — not to do it by hand in the console.

## Architecture context (the target platform)

- **Compute**: containers on **ECS Fargate (arm64 / Graviton)** — ~20% cheaper, no idle EC2. Async workers may use **Fargate Spot**. Avoid EKS at this stage (control-plane cost + ops overhead).
- **Edge**: CloudFront + **WAF** (TLS, per-tenant rate limiting) in front of **ALB** (app) and **S3** (static SPA).
- **Data**: **Aurora PostgreSQL Serverless v2** behind **RDS Proxy**; **ElastiCache (Redis/Valkey)** for hot reservations/locks.
- **Async**: **SQS** (FIFO where per-SKU order matters) for the transactional-outbox relay and background work.
- **Auth**: **Cognito** with a pre-token-generation Lambda that injects `tenant_id` + `tier` claims.
- **Networking**: private subnets for compute/data; reach AWS services via **VPC endpoints** to minimize NAT egress cost.

## Golden rules (do not violate)

1. **Everything is CDK; no click-ops.** Console changes are forbidden except break-glass, which must be backported to code immediately. Drift is a bug.
2. **Least-privilege IAM, scoped per service.** Grant the narrowest action/resource needed; use CDK `grant*()` helpers over hand-written wildcard policies. No `Action: "*"` / `Resource: "*"`.
3. **Tag every resource with `tenant`/`tier`/`env`/`service`/`cost-center`.** SaaS billing and noisy-neighbor detection depend on cost allocation tags being present from day one — retrofitting them is painful.
4. **Cost is a design constraint.** Prefer serverless/scale-to-low; justify any always-on resource. Every change states its monthly cost delta. Set AWS Budgets + alarms.
5. **Protect stateful resources.** Aurora, S3 data buckets, and similar carry `RemovalPolicy.RETAIN` (or deletion protection) in prod. A stack teardown must never be able to delete tenant data.
6. **Secrets only in Secrets Manager / Parameter Store**, injected at runtime — never in code, env files, or CDK source. No plaintext credentials anywhere in the repo.
7. **Pool vs Silo is config, not a fork.** Provisioning an Enterprise tenant to a dedicated DB is a parameterized path in the control-plane stack, not a copy-pasted bespoke stack.

## CDK structure & conventions

- **TypeScript CDK** (reuses the team's TS strength and shares types with the frontend repo where useful).
- Split by lifecycle into separate stacks: `network`, `data` (Aurora/Redis/Proxy), `app` (Fargate/ALB), `edge` (CloudFront/WAF/S3), `auth` (Cognito), `async` (SQS/workers), `controlplane` (tenant provisioning/routing), `observability`.
- One config object per environment (`dev`/`staging`/`prod`); no env-specific `if` scattered through constructs. Stateful and stateless stacks are deployed and reasoned about separately.
- Write reusable **constructs** for repeated patterns (e.g. a "tenant-tagged Fargate service"). Enforce tags via CDK Aspects so nothing ships untagged.

## Cost-control levers (bootstrap)

- Graviton everywhere it's supported; Fargate Spot for interruption-tolerant workers.
- Aurora Serverless v2 min-ACU low (and auto-pause for non-prod); right-size before scaling out.
- **VPC endpoints instead of NAT** for AWS-service traffic; NAT Gateways are a classic silent cost — justify each one.
- S3/CloudFront lifecycle policies; log retention limits on CloudWatch (logs are a sneaky line item).
- Consider Compute Savings Plans once usage is steady. Review the Cost Explorer breakdown by the tenant/tier tags regularly.

## Security & networking

- Compute and data live in **private subnets**; nothing stateful is public. Security groups are least-privilege and reference each other, not CIDR wildcards.
- WAF rules on the public edge; per-tenant throttling to contain noisy neighbors in the Pool.
- Rotate secrets; scope Cognito app clients and IAM roles per service.

## CI/CD & observability

- **GitHub Actions** → build/test → push image to **ECR** → `cdk deploy` per environment. Promote the same artifact across envs; `cdk diff` is part of the PR review.
- **OpenTelemetry → CloudWatch**; structured logs include `tenant_id`. Alarms on error rate, latency, queue depth, Aurora ACU/connections, and **cost budgets**. Define SLOs and alert on burn.

## Before you deploy — checklist

- [ ] Wrote a plan with the monthly cost delta and rollback; got sign-off for prod/IAM/network/data/cost changes.
- [ ] Ran `cdk diff` and confirmed it matches intent (no surprise replacements of stateful resources).
- [ ] IAM is least-privilege (no `*` action/resource); used `grant*()` where possible.
- [ ] All new resources carry `tenant`/`tier`/`env`/`service` cost-allocation tags (enforced by Aspect).
- [ ] Stateful resources have `RETAIN`/deletion protection in prod.
- [ ] No secrets in code; values come from Secrets Manager / Parameter Store.
- [ ] Serverless/scale-to-low chosen, or an always-on resource is explicitly justified; NAT use justified.
- [ ] Alarms/budgets cover the new resource; logs have a retention policy.
- [ ] Change is pure CDK — no undocumented console edits.

---

## Project standards (as built — Sprint 1)

These reflect the actual CI + local-dev setup.

- **CI:** `.github/workflows/ci.yml` has two jobs — `backend` (working-directory `backend`: setup-go, golangci-lint, build, test) and `frontend` (working-directory `frontend`: npm install, build). Green required to merge to `main`.
- **Local dev:** orchestrated by the **root `Makefile`** (delegates to `backend/` and `frontend/`); `make dev` runs both, `make setup` installs both. Postgres via `backend/docker-compose.yml`.
- **Backend image:** `backend/Dockerfile` — multi-stage → distroless, `linux/arm64` (Graviton). CDK deploys this same artifact (promote across envs; `cdk diff` in PR; per-tenant cost tags).

---

## Dependency governance

Every dependency is a long-term liability (security, maintenance, breakage, license, lock-in). The default answer to "should we add this library?" is **no** until it clears the checklist in `docs/Dependency-Governance.md`.

- The CDK app is TypeScript — same rules as the frontend: approved stack only, checklist + review for new deps, **commit `package-lock.json`, `npm ci`, `npm audit` in CI**, permissive licenses.
- Keep the CDK construct surface minimal; prefer `aws-cdk-lib` built-ins over third-party constructs unless they clearly pull their weight.
