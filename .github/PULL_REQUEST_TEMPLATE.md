## Summary

<!-- what & why -->

## Security checklist (required — see .claude/security)
- [ ] No new cross-tenant access; tenant-scoped changes go through `WithTenantTx` and have an isolation test
- [ ] Input validated; SQL parameterized (no string-built queries)
- [ ] Authorization enforced server-side (not just in the UI)
- [ ] No secrets or PII in code, config, or logs
- [ ] New dependencies cleared via `docs/Dependency-Governance.md` (scanned, licensed, allow-listed)
- [ ] Least privilege (IAM / DB roles / tokens)

## Tests
- [ ] Unit/integration added or updated; `make test` green
