---
name: cloud-infra-development
description: Best-practice standards for cloud infrastructure as code (primarily AWS with CDK/Terraform) — accounts and environments, least-privilege IAM, secrets, networking, cost control, CI/CD, and observability. Use this skill whenever creating or changing infrastructure, pipelines, IAM, or anything affecting cost or security posture. It encodes general, reusable cloud practices; project-specific stacks live in the repo.
---

# Cloud Infra Development

Reusable standards for infrastructure. Infra changes are slow to reverse and easy to make expensive or insecure — plan, and make everything code.

## Plan-first workflow

1. Restate the change and the environment(s)/resources affected. Production gets extra scrutiny.
2. Ask before assuming on cost impact, data-loss risk, blast radius, and scale.
3. Reuse existing stacks/constructs and conventions (naming, tagging, env config).
4. Write a short plan: resources changed, estimated monthly cost delta, IAM/security impact, rollback. Always run and read `diff` before applying. Sign-off for prod / IAM / networking / data stores / cost.
5. Apply, then verify (diff matched intent, tags present, alarms fire, no drift).

## Everything is code

- IaC only (CDK / Terraform); no click-ops. Console changes are break-glass and must be backported immediately. Drift is a bug.
- Split stacks by lifecycle (network / data / app / edge / observability); deploy stateful and stateless separately.
- One config object per environment (`dev`/`staging`/`prod`), ideally in separate accounts. Promote the **same build artifact** across environments — don't rebuild per env.

## Security & networking

- **Least-privilege IAM** — narrowest action/resource; prefer generated grants over hand-written wildcard policies. No `Action:"*"`/`Resource:"*"`.
- Secrets in a secret manager, injected at runtime; rotate; never in code or env files.
- Private subnets for compute/data; reach managed services via private endpoints to cut egress. TLS everywhere; encrypt at rest (KMS). Stateful resources get deletion protection / `RETAIN` in prod.

## Cost discipline

- Prefer serverless / scale-to-low; justify any always-on resource. State the monthly cost delta of each change. Set budgets + alarms.
- Tag everything (`env`, `service`, `owner`, `cost-center`, and `tenant`/`tier` for SaaS) for cost allocation.

## CI/CD & observability

- Pipeline: build/test → push artifact → migrate (gated) → deploy per env, with `diff` in PR review and green tests required.
- Structured logs, metrics, and traces; alarms on error rate, latency, saturation, and **cost budgets**; define SLOs and alert on burn.

## Before you deploy — checklist

- [ ] Plan with cost delta + rollback; signed off for prod/IAM/network/data/cost.
- [ ] `diff` reviewed; no surprise replacement of stateful resources.
- [ ] IAM least-privilege; secrets from the secret manager; no plaintext.
- [ ] Required tags present; stateful resources protected in prod.
- [ ] Alarms/budgets cover the change; change is pure IaC (no console edits).
