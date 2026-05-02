# 04 — Proposal Draft

> **Purpose:** GitHub issue or discussion draft for `mo-jaber/bsv-desktop` fork. Copy-paste ready.
> **Tone:** Maintainer-to-maintainer. No marketing language.

---

## Title Options

1. **UI/UX uplift: MUI to Tailwind + shadcn/ui**
2. **Proposal: migrate presentation layer from MUI to Tailwind CSS + shadcn/ui**
3. **Frontend modernization — Tailwind + shadcn/ui migration**

---

## Summary

Proposing a phased migration of the bsv-desktop presentation layer from MUI Material v6 (with Emotion CSS-in-JS) to Tailwind CSS + shadcn/ui. The wallet logic, storage, IPC, and Electron infrastructure remain untouched. The goal is better design control, smaller bundle, and a maintainable styling foundation — without disrupting any wallet behavior.

---

## Motivation

The current UI stack creates a ceiling on design quality and developer experience:

- **MUI defaults dominate the visual identity.** The wallet looks like a Material Design app with a navy palette applied. Achieving distinctive, wallet-appropriate design requires fighting MUI's opinions rather than building on them.
- **Pre-existing maintenance debt compounds the problem.** react-router-dom v5 (end-of-life), no linting, TypeScript `strict: false`, no test coverage, and partially dead styling code (`navigation/style.ts` references undefined theme keys that resolve to `undefined` at runtime). These aren't blockers individually, but together they make confident changes difficult.
- **Emotion CSS-in-JS adds runtime cost and complexity.** `makeStyles`, `styled()`, and `sx` props are scattered across components with no consistent pattern. Some components use inline styles, others use theme templates, others use `sx`. A utility-first system like Tailwind eliminates this inconsistency.
- **Component library lock-in.** Every MUI component carries opinions about DOM structure, animation, and accessibility that may or may not match wallet UX needs. shadcn/ui provides copy-in-place primitives that can be modified without forking a library.

A full audit has been completed (see `docs/migration/01-design-system-context.md` and `docs/migration/02-behavior-anchors.md`) documenting every route, component, theme token, and behavioral contract in the current codebase.

---

## Non-Goals

Explicitly **not** changing:

- Wallet logic (`@bsv/wallet-toolbox`, `@bsv/sdk`, WalletContext business logic)
- Snapshot format (Version 3 binary)
- Storage layer (SQLite, Knex, `electron/storage.ts`)
- HTTP server (Express on ports 2121/3321)
- IPC handlers and preload bridge
- Monitor worker process
- Electron main process or auto-updater
- Build and packaging configuration
- Encryption, key derivation, HD paths
- Feature additions — this is a reskin, not a feature release

---

## Approach

Phased PR sequence. Each PR is independently reviewable and deployable. MUI and Tailwind coexist during migration; MUI is removed only in the final PR.

### PR 1: Foundation

- Add ESLint + Prettier configuration
- Enable TypeScript `strict: true` and fix type errors
- Add Vitest test infrastructure (unit tests for critical utilities)
- Update port documentation (both docs should reference both 2121 HTTPS and 3321 HTTP)
- Clean up dead code flagged in audit (LostPassword.tsx, LostPhone.tsx — pending confirmation)

### PR 2: Router Migration

- react-router-dom v5 → react-router v7
- Replace `RouteComponentProps`, `withRouter`, `useHistory()` with v7 equivalents (`useNavigate()`, `useParams()`, `useLocation()`)
- Remove `@types/react-router-dom` dev dependency
- All behavior anchors must pass (navigation in/out for every route)

### PR 3: Tailwind + shadcn Infra + Reference Screen

- Install Tailwind CSS, configure `tailwind.config.ts` with design tokens mapped from current MUI theme
- Copy shadcn/ui primitives (Button, Dialog, Input, Tabs, etc.)
- Replace react-toastify with sonner
- Replace @mui/icons-material with lucide-react
- Migrate one reference screen (Greeter) end-to-end as proof of pattern
- MUI still present in other screens — dual styling systems coexist

**Note:** An exploratory design spike should precede this PR to establish the visual direction (palette, spacing, component feel). The audit provides the raw material; Claude Design (Step 2) produces the design tokens and reference mockups.

### PRs 4–N: Screen-by-Screen Migration

Suggested order (simplest screens first, building pattern confidence):

1. AppCatalog (standalone, no context deps)
2. Apps (list + pagination)
3. Payments (4-tab, moderate complexity)
4. Settings (config UI, multiple sections)
5. MyIdentity (certificates, key reveal)
6. Trust (navigation blocking, unsaved state)
7. Security (two-view architecture)
8. LegacyBridge (two-tab, external API)
9. Recovery pages (3 routes)
10. Access pages (AppAccess, BasketAccess, ProtocolAccess, CounterpartyAccess, CertificateAccess — similar structure)
11. Permission handler modals (9 handlers — batch or individually)
12. Dashboard shell + Menu (sidebar, profile management)

Each PR:
- Migrates one screen (or logical group) from MUI to Tailwind + shadcn
- Passes all behavior anchors for that route (per `02-behavior-anchors.md`)
- Dark mode works
- All `t()` keys render
- No MUI imports in migrated files

### Final PR: MUI Removal

- Remove `@mui/material`, `@mui/icons-material`, `@mui/styles`
- Remove `@emotion/react`, `@emotion/styled`
- Remove `src/lib/theme.d.ts`, `src/lib/components/Theme.tsx`, `navigation/style.ts`
- Verify zero MUI/Emotion imports remain
- Bundle size comparison (before/after)

---

## Open Questions

1. **Dead code cleanup:** Should `LostPassword.tsx` and `LostPhone.tsx` be deleted in PR 1, or are they intended for future completion?
2. **Brand direction:** Current navy `#1B365D` as starting point — any constraints on palette exploration?
3. **PR cadence:** How frequently can you review? One screen per PR means ~15 PRs. Could batch related screens (e.g., all access pages in one PR) to reduce count to ~8-10.
4. **Port documentation:** Both ports (2121 HTTPS, 3321 HTTP fallback) are correct. Should README reference both?

---

## What I'm Asking For

Directional sign-off. Not line-by-line approval — just:

- Does a MUI → Tailwind + shadcn migration make sense for this project?
- Is the phased PR approach acceptable (vs. big-bang)?
- Any hard constraints I should know about before starting?

The audit documentation (`docs/migration/01-design-system-context.md` through `03-design-constraints.md`) is ready for review and provides the foundation for the actual migration work.

---

## Risks Acknowledged

| Risk | Mitigation |
|------|-----------|
| Dual styling systems during migration | Each PR migrates complete screens; coexistence is temporary. No screen is half-MUI half-Tailwind. |
| Migration drift (screens diverge in pattern) | Reference screen (Greeter) establishes pattern; design constraints doc (`03-design-constraints.md`) enforces consistency. |
| shadcn copy maintenance | shadcn components are copied into project. Updates are manual. This is by design — full control — but means tracking upstream fixes. |
| Router migration breaks navigation | Behavior anchors document every route-in/route-out. Automated or manual verification against the anchor list. |
| TypeScript strict mode reveals hidden issues | Better to find them now. Issues fixed in PR 1 before any UI work begins. |
| Bundle size during coexistence | Temporary. MUI + Tailwind both loaded during migration phase. Resolved in final PR. |
