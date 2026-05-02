# 03 — Design Constraints

> **Audience:** Claude Design (Step 2) and Claude Code (Step 3)
> **Purpose:** Hard rules. Paste verbatim into context windows. No deviation.

---

## 1. Hard Constraints

- **UI framework:** Tailwind CSS + shadcn/ui (components copied into project, not installed as dependency).
- **Icons:** lucide-react. No MUI icons.
- **Toasts:** sonner. No react-toastify.
- **Router:** react-router v7. No react-router-dom v5. No `RouteComponentProps`, no `withRouter`, no `useHistory()`. Use `useNavigate()`, `useParams()`, `useLocation()`.
- **i18n:** All user-facing strings via `t()`. Reuse existing keys. New keys follow `category_descriptor` convention. No hardcoded English.
- **TypeScript:** `strict: true`. All new code fully typed.
- **Dependencies:** No new runtime dependencies without explicit approval. shadcn/ui primitives, lucide-react, sonner, and Tailwind are the only additions.
- **Behavior anchors:** Every route's data consumed, actions emitted, conditional behavior, and edge cases documented in `02-behavior-anchors.md` are non-negotiable. A migrated screen passes when all anchors hold.
- **Dark mode:** Must work. 3-mode (light / dark / system). localStorage key `userTheme`.
- **MUI removal:** Post-migration, zero MUI / Emotion / CSS-in-JS imports remain. No `@mui/*`, no `@emotion/*`, no `makeStyles`, no `styled()`, no `sx` prop.

---

## 2. Brand Direction (Provisional)

- Current navy `#1B365D` is the starting point, not locked. Design may explore adjacent palettes.
- **Avoid** generic shadcn gray defaults. The wallet must look intentional, not like a template.
- Final HSL token values TBD after design exploration.
- Approval colors (`#86c489`, `#96c486`, `#86a7c4`, `#ad86c4`) are provisional — may be adjusted for contrast and accessibility.
- Minimum contrast ratio: WCAG AA (4.5:1 for normal text, 3:1 for large text).

---

## 3. Wallet-Domain UX Rules

| Rule | Rationale |
|------|-----------|
| Confirmation dialogs default to cancel (not confirm) | Prevent accidental irreversible actions |
| Destructive actions require explicit confirmation | Delete profile, revoke access, sweep funds |
| Secrets hidden by default, revealed on user action | Mnemonic, hex key, recovery key — always behind a reveal button |
| Amounts displayed unambiguously | Always show unit (sats or fiat), never bare numbers |
| Addresses and keys: copyable + truncated with tooltip | Full value in tooltip or on click; truncated in display |
| Transaction status visually distinguished | Distinct color/badge for: completed, sending, failed, unsigned, nosend, nonfinal, unproven |
| Permission requests always show originator | User must know which app is asking |
| Loading states on every async operation | No silent spinners; every pending state visible |
| Error feedback on every failed operation | Toast or inline — never silent swallow |

---

## 4. Design Principles

1. **Restraint over emphasis.** Not everything needs a card, shadow, or gradient. Use whitespace to separate. Reserve visual weight for actions and alerts.

2. **Progressive disclosure.** Show summary first, details on demand. Collapse secondary sections. Don't front-load every field.

3. **Type-driven differentiation.** Use font weight, size, and opacity to establish hierarchy — before reaching for color or borders.

4. **Date grouping.** Transaction lists grouped by date (Today, Yesterday, earlier). Not flat infinite scrolls.

5. **Designed empty states.** Every list and page has a purposeful empty state — not just "No items found." Include context and next-action hints.

6. **Consistent copy patterns.** Every copyable value: click → clipboard → brief visual confirmation (check icon, 2s) → revert. Same pattern everywhere.

7. **Motion with purpose.** Transitions for state changes (accordion expand, dialog enter/exit, tab switch). No decorative animation. No page-level transitions.

8. **Density control.** Dashboard pages should not feel cramped. Permission dialogs should not waste space. Match density to task urgency.

---

## 5. Out of Scope

The following are **not touched** by this migration. Do not modify, redesign, or refactor:

- Wallet logic (`@bsv/wallet-toolbox`, `@bsv/sdk` usage, WalletContext business logic)
- Database layer (SQLite, Knex, `electron/storage.ts`)
- Express HTTP server (`electron/httpServer.ts`)
- IPC handlers and preload bridge (`electron/main.ts`, `electron/preload.ts`)
- Monitor worker process (`electron/monitor-worker.ts`)
- Electron main process lifecycle
- Auto-updater (`electron/updater.cjs`)
- Build and packaging configuration (`electron-builder`)
- Snapshot format (Version 3 binary format)
- Encryption, key derivation, HD paths

---

## 6. Review Checklist (Step 3 Handoff)

Before marking any screen migration complete, verify:

- [ ] All behavior anchors from `02-behavior-anchors.md` pass for this route
- [ ] Dark mode renders correctly (no hardcoded colors, all tokens resolve)
- [ ] All `t()` keys render (no missing translations, no raw keys displayed)
- [ ] No MUI imports remain in the migrated file
- [ ] No Emotion imports remain in the migrated file
- [ ] Loading, empty, and error states all render
- [ ] Permission dialogs still trigger and resolve correctly
- [ ] Navigation in and out of the screen works (back button, deep links, redirects)
