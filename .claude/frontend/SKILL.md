---
name: react-development
description: Best-practice standards for building React applications in TypeScript — component design, server/client state, typed API access, forms, performance, accessibility, styling, PWA/offline, testing, and frontend security. Use this skill whenever writing, reviewing, or refactoring React/TypeScript UI code. It encodes general, reusable React practices; project-specific facts live in the repo's CLAUDE.md.
---

# React Development

Reusable standards for React + TypeScript apps. Optimize for clarity, type-safety, and a fast, accessible UI.

## Plan-first workflow

1. Restate the screen/feature and who uses it (and on what device, if it matters).
2. Ask before assuming on data shape, loading/empty/error states, and offline expectations.
3. Find the nearest existing component/hook and match its conventions.
4. Write a short plan for non-trivial work (components, data flow, states); get sign-off.
5. Implement, then verify against the checklist (incl. a11y + error/empty/loading states).

## Stack & structure

- React + TypeScript, built with Vite (or the project's chosen framework). Strict TS (`strict: true`).
- Function components + hooks only. Keep components small and focused; lift state only as far as needed.
- Co-locate component, styles, and tests; share logic via custom hooks, not copy-paste.

## State management

- **Server state** via TanStack Query (caching, retries, mutations). Cache keys include any tenant/scope context; invalidate the right keys on mutation success. Don't duplicate server data into local state.
- **Local/UI state** via `useState`/`useReducer`/context. Avoid prop-drilling with composition or context; reach for a global store only when genuinely shared.

## Typed API access

- All backend calls go through a **single typed client** (one `request<T>()` wrapper) — never hand-roll `fetch` in components. Generate types from the API's OpenAPI spec where possible.
- Centralize auth/headers, base URL (from env), and error handling in that client.

## Forms, performance, accessibility

- Forms: React Hook Form + Zod (validate at the edge; reuse schemas).
- Performance: route-based code splitting; memoize only when measured; virtualize long lists; keep bundles lean.
- Accessibility: semantic HTML, labels, keyboard support, sufficient contrast; prefer accessible primitives (e.g. Radix/shadcn). Handle loading/empty/error states explicitly.

## Styling

- Use one styling system consistently (e.g. Tailwind / a component library). Don't mix paradigms.

## PWA / offline (when needed)

- Service worker for app-shell caching; queue state-changing actions (with an idempotency key) and sync on reconnect; make sync status visible. Trust the server as source of truth on conflict.

## Testing & security

- Vitest + React Testing Library for components/hooks (mock the API client); Playwright for critical flows.
- Never put secrets in the bundle; encode/sanitize rendered output (avoid XSS); rely on the auth token, not client state, for identity; set security headers/CSP at the edge.

## Before you commit — checklist

- [ ] Plan written + signed off for non-trivial work.
- [ ] All API calls go through the typed client; types match the backend.
- [ ] Loading / empty / error (and offline, if applicable) states handled and visible.
- [ ] Optimistic updates roll back on failure.
- [ ] Accessible (semantic, keyboard, contrast); strings externalized if i18n is used.
- [ ] No secrets in the bundle; output encoded; `tsc --noEmit` clean.
