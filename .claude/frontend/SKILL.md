---
name: wms-frontend-development
description: Engineering standards for the WMS SaaS frontend — a TypeScript + React (Vite) PWA that runs on both office desktops and rugged Android RF handheld scanners. Use this skill whenever writing, reviewing, or refactoring ANY frontend code for this project — components, screens, the API client, state/data fetching, scanner input handling, offline/sync logic, or auth/tenant context. Consult it even for small UI changes, because this app has non-obvious correctness rules (offline-tolerant action queue with idempotency keys, hardware barcode-scanner input, multi-tenant context from JWT, and warehouse-grade ergonomics) that are easy to break silently. If a change touches scanning, offline behavior, tenant data, or operator workflows, this skill applies.
---

# WMS Frontend Development

Standards for the WMS SaaS frontend. The goal is **one TypeScript/React codebase** that serves both the office/admin web UI and the warehouse-floor handheld UI, works on cheap rugged Android devices over flaky warehouse Wi‑Fi, and never lets a tenant see another tenant's data.

## Plan-first workflow (do this before writing code, every time)

Warehouse software is operational: a confusing screen or a dropped scan stops a picker mid-aisle, and a wrong optimistic update can show stock that isn't there. Rework on a live floor is expensive. So invest two minutes up front:

1. **Restate the task** in one or two sentences and identify the operator + device it runs on (desktop admin vs handheld picker). The constraints differ wildly.
2. **Ask before assuming** when anything material is unclear — which workflow step, online/offline expectations, what the screen must show when the network is down, whether the action is state-changing. Prefer one good question now over a wrong build.
3. **Inspect existing patterns first.** Find the nearest existing screen/component/hook and match its conventions (data fetching, error states, scanner handling). Consistency beats cleverness.
4. **Write a short plan** before implementing: files to add/touch, the data this screen reads/writes, online + offline behavior, and any risk (oversell display, double-submit, tenant leakage). Share it and get a thumbs-up for anything beyond a trivial change.
5. **Implement, then verify** against the checklist at the end — including on a simulated handheld viewport and an offline toggle.

If the plan reveals a backend or schema change is also needed, stop and flag it rather than working around it on the client.

## Architecture context

- **Vite + React + TypeScript** SPA, shipped as static assets to **S3 + CloudFront** (no SSR server to run — cheaper, simpler, fits the bootstrap cost posture). SEO is irrelevant for an authed internal app.
- **PWA**: installable, with a service worker for app-shell caching and an offline action queue. Warehouse Wi‑Fi has dead zones; the app must degrade, not die.
- Talks to the Go backend over REST/JSON. Auth via **Cognito** JWT; the token carries `tenant_id` and `tier`.
- Runs on two form factors from one codebase: **desktop** (admin, configuration, dashboards) and **handheld** (receiving, putaway, pick, pack, count) — routed by role/route, styled responsively.

## Golden rules (do not violate)

1. **Never trust the client for tenant isolation, but never leak it either.** The backend enforces isolation via RLS, but the UI must still scope every request/cache key by the active tenant so a tenant switch can't show stale data from another tenant.
2. **Every state-changing request carries an `Idempotency-Key`.** Handhelds replay queued actions on reconnect; without a stable key a re-sent "pick 5 units" applies twice. Generate the key when the user acts, not when the request is sent.
3. **Optimistic UI must be reversible and honest.** You may show an action as pending, but on failure you must roll back and tell the operator clearly. Never display derived stock as confirmed before the server accepts the movement.
4. **Scanner input is the primary input on handhelds.** Screens used on the floor must accept hardware scanner input (keyboard-wedge) without a focused text field being a prerequisite, and must not depend on a soft keyboard.
5. **Fail loud on the floor.** Errors must be unmissable (large, high-contrast, optionally audible/haptic). A silent failure in a warehouse means mis-shipped goods.
6. **Type-safety end to end.** API types are generated from the backend's OpenAPI spec — never hand-write request/response shapes that can drift from the server.

## Library stack (use these; do not add alternatives without discussion)

- Build/dev: **Vite**
- UI: **React + TypeScript**, **Tailwind CSS**, **shadcn/ui** (Radix primitives — accessible by default)
- Routing: **TanStack Router** (type-safe routes)
- Server state: **TanStack Query** (caching, retries, offline-aware mutations)
- Client/form state: React state + **React Hook Form**; validation with **Zod** (reuse schemas for runtime validation of API payloads)
- API client: generated from OpenAPI (e.g. `openapi-typescript` + a thin typed fetch wrapper)
- PWA/offline: **Workbox** service worker + **IndexedDB** (via `idb`) for the action queue and cached reference data
- Tests: **Vitest** + **React Testing Library**; **Playwright** for critical operator flows
- i18n: **react-i18next** (Thai + English from day one)

## Scanner & handheld ergonomics

Rugged Android scanners (Zebra, Honeywell, etc.) run Chrome and emit a scan as fast keystrokes ending in Enter — so a web app "just works" if you handle input correctly:

- Capture scans with a **global keydown listener / scan buffer** that assembles characters arriving faster than human typing and fires on the terminator. Don't require the operator to tap an input first.
- Design for **gloves, glare, and one hand**: large touch targets (min ~48px), high contrast, minimal typing, primary action reachable by thumb.
- Confirm each scan with **visual + audio + haptic** feedback so the operator knows it registered without staring at the screen.
- Keep screens **single-purpose and linear** (scan location → scan item → enter qty → confirm). Don't port a dense desktop form to a handheld.

## Offline & sync (PWA)

The floor must keep working when Wi‑Fi drops:

- Cache **reference/read data** (bin maps, SKU lookups for the active session) in IndexedDB so lookups don't block on the network.
- Queue **state-changing actions** in IndexedDB with their `Idempotency-Key`, show them as "pending sync", and flush in order on reconnect via TanStack Query's mutation/retry mechanics. The backend is idempotent on the key, so replays are safe.
- Make sync **observable**: a persistent indicator of online/offline and the number of unsynced actions. Never let the operator believe work is saved when it's only local.
- Resolve conflicts by **trusting the server** (it is the source of truth for stock) and surfacing rejected actions for the operator to redo.

## Multi-tenant in the client

- Read `tenant_id`/`tier` from the verified JWT; include the tenant in **every query cache key** and clear/segment caches on tenant switch.
- Gate features by `tier` (e.g. Enterprise-only screens) from a single capabilities source, not scattered conditionals.
- Never hardcode tenant- or environment-specific URLs; resolve API base from config injected at build/deploy.

## Performance (target: cheap Android handhelds)

- Route-based **code splitting**; keep the handheld bundle lean — pickers' devices are low-RAM.
- Virtualize long lists (orders, SKUs). Avoid heavy client-side computation; let the backend do aggregation.
- Measure on a throttled mid-tier device profile, not your laptop.

## Before you commit — checklist

- [ ] Wrote a short plan and (for non-trivial work) got sign-off before coding.
- [ ] Every mutation sends an `Idempotency-Key`; queued offline actions reuse it on replay.
- [ ] Query cache keys include the active `tenant_id`; tenant switch can't show stale data.
- [ ] Optimistic updates roll back and surface errors visibly on failure.
- [ ] Floor screens accept hardware scanner input without a pre-focused field or soft keyboard.
- [ ] Error/empty/loading and **offline** states are all handled and visible.
- [ ] API types come from the generated OpenAPI client (no hand-written shapes).
- [ ] Verified on a handheld-sized viewport and with the network toggled offline.
- [ ] Strings are translated (th/en), not hardcoded.

---

## Project standards (as built — Sprint 1)

These reflect the actual scaffold. Follow them so new work matches what exists.

- **Location:** the frontend lives in `frontend/`; run `npm` there (or root `make run-frontend`).
- **Pinned stack:** Vite + React + TypeScript, Tailwind, TanStack Query, `vite-plugin-pwa`. No alternatives without discussion.
- **API client:** all backend calls go through the single `request<T>()` wrapper in `src/lib/api.ts` — it injects `content-type` + `X-Tenant-Id` and throws on non-2xx. Never hand-roll `fetch` in components; add typed functions to `api.ts`.
- **Config:** Vite env — `VITE_API_URL`, `VITE_TENANT_ID` (see `.env.example`). The tenant header is a Sprint 1 stub; Sprint 2 swaps it for a Cognito JWT.
- **Server state:** use TanStack Query (`useQuery`/`useMutation`); invalidate the relevant `queryKey` on mutation success. Cache keys carry tenant context once tenant-switching exists.
- **Scripts:** `npm run dev` · `npm run build` (`tsc -b && vite build`) · `npm run lint` (`tsc --noEmit` typecheck).
- **CI:** the `frontend` job runs npm install → build.

---

## Dependency governance

Every dependency is a long-term liability (security, maintenance, breakage, license, lock-in). The default answer to "should we add this library?" is **no** until it clears the checklist in `docs/Dependency-Governance.md`.

- Stay on the **approved stack**. Anything new requires the add-a-dependency checklist **and** PR review.
- Prefer a built-in / existing dep over a new one; **avoid micro-packages** for trivial things. **One library per concern.**
- **Pin versions; commit `package-lock.json`** and use `npm ci` in CI. Never `@latest`/floating ranges that drift.
- Permissive licenses only; watch transitive + bundle size (a dep is also shipped to the handheld).
- **Enforced in CI:** `npm audit --audit-level=high`; consider a bundle-size budget.
