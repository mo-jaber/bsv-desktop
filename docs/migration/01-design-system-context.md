# 01 — Design System Context

> **Audience:** Claude Design (Step 2) and Claude Code (Step 3)
> **Purpose:** Prime the design agent on the existing system so it can produce accurate, migration-safe designs.

---

## 1. Stack Snapshot

| Layer | Current | Target | Notes |
|-------|---------|--------|-------|
| UI framework | MUI Material v6.4.8 + Emotion | Tailwind CSS + shadcn/ui (copied) | Migration |
| Icons | @mui/icons-material v6.4.8 | lucide-react | Migration |
| Toasts | react-toastify v11.0.5 | sonner | Migration |
| Router | react-router-dom v5.2.0 | react-router v7 | Migration |
| TypeScript | ~5.6.2, `strict: false` | same version, `strict: true` | Config change |
| Lint | no-op (`echo 'No linting'`) | ESLint + Prettier | Addition |
| React | 18.3.1 | 18.3.1 | Frozen |
| Bundler | Vite 6.3.6 | Vite 6 | Frozen |
| Electron | 41.2.0 | 41.2.0 | Frozen |
| Wallet core | @bsv/wallet-toolbox ^2.1.19, @bsv/sdk ^2.0.13 | same | Frozen |
| DB | better-sqlite3 ^12.8.0 + Knex ^3.1.0 | same | Frozen |
| HTTP server | Express on 2121 (HTTPS) / 3321 (HTTP fallback) | same | Frozen |
| Worker | monitor-worker.ts (forked Node process) | same | Frozen |

### Explicit ban list

Post-migration, **none** of the following may remain:
- `@mui/material`, `@mui/icons-material`, `@mui/styles`
- `@emotion/react`, `@emotion/styled`
- Any other CSS-in-JS library (styled-components, Linaria, etc.)
- `makeStyles`, `styled()`, `sx` prop usage

---

## 2. Theme & Design Tokens (Current MUI)

Source: `src/lib/components/Theme.tsx`, `src/lib/theme.d.ts`

### Palette

**Light mode:**
| Token | Value |
|-------|-------|
| `primary.main` | `#1B365D` |
| `secondary.main` | `#2C5282` |
| `background.default` | `#FFFFFF` |
| `background.paper` | `#F6F6F6` |
| `text.primary` | `#4A4A4A` |
| `text.secondary` | `#4A5568` |

**Dark mode:**
| Token | Value |
|-------|-------|
| `primary.main` | `#FFFFFF` |
| `secondary.main` | `#487dbf` |
| `background.default` | `#1D2125` |
| `background.paper` | `#1D2125` |
| `text.primary` | `#FFFFFF` |
| `text.secondary` | `#888888` |

### Approval colors (custom augmentation)

Source: `src/lib/theme.d.ts` — augments `Theme` with `approvals` object.

| Token | Value | Usage |
|-------|-------|-------|
| `approvals.protocol` | `#86c489` | Protocol permission dialogs |
| `approvals.basket` | `#96c486` | Basket permission dialogs |
| `approvals.identity` | `#86a7c4` | Identity permission dialogs |
| `approvals.renewal` | `#ad86c4` | Renewal permission dialogs |

### Typography

```
fontFamily: '"Helvetica","Arial",sans-serif'
```

| Level | Weight | Size | Responsive (<=900px) |
|-------|--------|------|---------------------|
| h1 | 700 | 2.5rem | 1.8rem |
| h2 | 700 | 1.7rem | 1.6rem |
| h3 | — | 1.4rem | — |
| h4 | — | 1.25rem | — |
| h5 | — | 1.1rem | — |
| h6 | — | 1rem | — |

### Spacing

Base unit: **8px** (`theme.spacing(1) = 8px`)

### Shape

`borderRadius: 2` (global default)

### Custom template tokens

Source: `Theme.tsx` lines 355–398 — augments `Theme` with `templates` object.

| Token | Value | Usage |
|-------|-------|-------|
| `templates.page_wrap` | `maxWidth: min(1440px, 100vw)`, `margin: auto`, `padding: 56px` | Page container |
| `templates.subheading` | `textTransform: uppercase`, `letterSpacing: 6px`, `fontWeight: 700` | Section headers |
| `templates.boxOfChips` | `display: flex`, `justifyContent: left`, `flexWrap: wrap`, `gap: 8px` | Chip grid layout |
| `templates.chip(size, bg)` | `height: size*32px`, `borderRadius: 16px`, `padding: 8px`, `margin: 4px` | Chip card base |
| `templates.chipLabel` | `display: flex`, `flexDirection: column` | Chip label container |
| `templates.chipLabelTitle(size)` | `fontSize: max(size*0.8, 0.8)rem`, `fontWeight: 500` | Chip title text |
| `templates.chipLabelSubtitle` | `fontSize: 0.7rem`, `opacity: 0.7` | Chip subtitle text |
| `templates.chipContainer` | `position: relative`, `display: inline-flex`, `alignItems: center` | Chip wrapper |

### Component overrides

| Component | Override | Light | Dark |
|-----------|----------|-------|------|
| `MuiCssBaseline` body | background | `#FFFFFF` + subtle gradient `rgba(27,54,93,0.05)` | `#1D2125` + gradient `rgba(27,54,93,0.1)` |
| `MuiButton` contained | bg / color | `#1B365D` / `#FFFFFF` | `#FFFFFF` / `#1B365D` |
| `MuiButton` outlined | border / color | `#1B365D` | `#FFFFFF` |
| `MuiButton` disabled | bg | `rgba(0,0,0,0.12)` | `rgba(255,255,255,0.12)` |
| `MuiButton` | textTransform | `none` (both modes) | — |
| `MuiPaper` | backgroundImage | `none` (both) | — |
| `MuiAppBar` | bg | `#1B365D` | `#1D2125` |
| `MuiCard` | borderRadius / border | `12` / `rgba(0,0,0,0.12)` | `12` / `rgba(255,255,255,0.12)` |
| `MuiChip` | borderRadius | `8` | `8` |
| `MuiDialog` paper | borderRadius / bg | `8` / `#FFFFFF` | `8` / `#1D2125` |
| `MuiDialogTitle` | bg / color | `#1B365D` / `#FFFFFF` | `#1D2125` / `#FFFFFF` |
| `MuiDialogContent` | bg / color | `#FFFFFF` / `#4A4A4A` | `#1D2125` / `#FFFFFF` |
| `MuiDialogActions` | bg / border-top | `#F6F6F6` / `rgba(0,0,0,0.12)` | `#1D2125` / `rgba(255,255,255,0.12)` |

### Dark mode implementation

- **3-mode support:** light / dark / system
- **Persistence:** `localStorage.getItem('userTheme')` — values: `'light'`, `'dark'`, `'system'`
- **OS detection:** `useMediaQuery('(prefers-color-scheme: light)')` (note: checks for light, inverted logic)
- **Sync:** WalletContext `settings?.theme?.mode` ↔ localStorage, with version counter to force useMemo re-run
- **Provider nesting:** `StyledEngineProvider injectFirst` → `ThemeProvider` → `CssBaseline`

> **Note:** Current navy `#1B365D` is a starting point for design exploration, not locked.

### AMBIGUOUS: Undefined theme keys in `navigation/style.ts`

`navigation/style.ts` references the following keys **not defined** in `createTheme()`:

- `theme.palette.background.mainSection` — used in `content_wrap`
- `theme.palette.background.leftMenu` — used in `list_wrap`
- `theme.palette.background.leftMenuHover` — used in list item hover
- `theme.palette.background.leftMenuSelected` — used in selected list item
- `theme.palette.background.scrollbarTrack` — used in scrollbar styling
- `theme.palette.background.scrollbarThumb` — used in scrollbar styling
- `theme.maxContentWidth` — used in `page_container`

These resolve to `undefined` at runtime. The Menu component has been rewritten to use inline `sx` props on the Drawer instead, making `list_wrap` dead code. However, `content_wrap` and `page_container` are still consumed by `Dashboard/index.tsx` via `makeStyles`. Impact: background colors fall through to defaults; scrollbar styling is invisible.

---

## 3. Component Inventory (Current MUI)

Components used 2+ times, classified by migration complexity.

### Classification key

- **VISUAL** — Presentation only, direct shadcn mapping exists
- **BEHAVIORAL** — Contains business logic, API calls, or complex state; needs careful migration
- **CONTAINER** — Logic-heavy wrapper; visual layer is thin

### Reusable components

| Component | File | Props (verbatim from source) | Purpose | Classification | shadcn Equivalent |
|-----------|------|-----|---------|----------------|-------------------|
| AmountDisplay | `src/lib/components/AmountDisplay/index.tsx` | `{ abbreviate?, showPlus?, description?, children, showFiatAsInteger? }` | Currency display with sats/fiat toggle via ExchangeRateContext | BEHAVIORAL | Custom (no equivalent) |
| AppChip | `src/lib/components/AppChip/index.tsx` | `extends RouteComponentProps { label, showDomain?, clickable?, size?, onClick?, backgroundColor?, expires?, onCloseClick? }` | App metadata card with favicon fetch | BEHAVIORAL | Custom card variant |
| BasketChip | `src/lib/components/BasketChip/index.tsx` | `extends RouteComponentProps { basketId, lastAccessed?, domain?, clickable?, size?, onClick?, expires?, onCloseClick?, canRevoke? }` | Basket metadata card | BEHAVIORAL | Custom card variant |
| CertificateChip | `src/lib/components/CertificateChip/index.tsx` | `extends RouteComponentProps { certType, expiry?, certVerifier, canRevoke?, onRevokeClick?, clickable?, size?, backgroundColor?, onClick?, alldetails? }` | Certificate info card | BEHAVIORAL | Custom card variant |
| CounterpartyChip | `src/lib/components/CounterpartyChip/index.tsx` | `extends RouteComponentProps { counterparty, clickable?, size?, onClick?, expires?, onCloseClick?, canRevoke?, label? }` | Identity card with avatar | BEHAVIORAL | Custom card variant |
| ProtoChip | `src/lib/components/ProtoChip/index.tsx` | `extends RouteComponentProps { securityLevel, protocolID, counterparty?, lastAccessed?, originator?, clickable?, size?, onClick?, expires?, onCloseClick?, canRevoke?, description?, iconURL?, backgroundColor? }` | Protocol metadata card | BEHAVIORAL | Custom card variant |
| PageHeader | `src/lib/components/PageHeader/index.tsx` | `{ title, subheading, icon, buttonTitle, buttonIcon?, onClick, history, showButton?, showBackButton?, onBackClick? }` | Page header with optional back button and action button | VISUAL | Custom header |
| CustomDialog | `src/lib/components/CustomDialog/index.tsx` | `extends DialogProps { title, children, description?, actions?, minWidth?, icon? }` | Modal wrapper with themed title bar | VISUAL | Dialog (shadcn) |
| PlaceholderAvatar | `src/lib/components/PlaceholderAvatar/index.tsx` | `{ name, variant?, size?, sx? }` | Deterministic color avatar from name hash | VISUAL | Avatar (shadcn) |
| AppLogo | `src/lib/components/AppLogo.tsx` | `{ className?, size?, color?, rotate? }` | SVG logo with optional spin animation | VISUAL | Custom SVG |
| Action | `src/lib/components/Action.tsx` | `{ txid, description, amount, inputs, outputs, fees?, onClick?, isExpanded? }` | Transaction detail accordion row | VISUAL | Collapsible (shadcn) |
| WalletConfig | `src/lib/components/WalletConfig.tsx` | `{ autoExpand?, hideLoginType?, open?, onToggle? }` | Network/auth/storage configuration panel | CONTAINER | Custom (logic-heavy) |
| MessageBoxConfig | `src/lib/components/MessageBoxConfig/index.tsx` | `{ showTitle?, embedded? }` | Message box URL setup | CONTAINER | Custom (logic-heavy) |
| ErrorBoundary | `src/lib/components/ErrorBoundary.tsx` | `extends WithTranslation { children, fallback? }` | React error boundary | CONTAINER | Keep as-is |
| Profile | `src/lib/components/Profile.tsx` | none (context-driven) | Sidebar balance display with paginated listOutputs | BEHAVIORAL | Custom (balance + paginated fetch) |
| PageLoading | `src/lib/components/PageLoading/index.tsx` | none | Full-page loading spinner | VISUAL | shadcn Skeleton or Loader2 |

> **Note on RouteComponentProps:** All Chip components extend react-router-dom v5's `RouteComponentProps` to access `history.push()`. During the router v5→v7 migration, these must be refactored to use `useNavigate()` hook instead. The `history` prop dependency must be fully removed.

---

## 4. Layout Shell

### Provider nesting order (outermost → innermost)

Source: `src/lib/UserInterface.tsx`

```
LanguageProvider
  └─ UserContextProvider (nativeHandlers, appVersion, appName)
      └─ WalletContextProvider (onWalletReady, permissionModules)
          └─ AppThemeProvider
              └─ ExchangeRateContextProvider
                  └─ HashRouter
                      ├─ AuthRedirector (invisible redirect component)
                      ├─ BreakpointProvider (xs/sm/md/or queries)
                      │   ├─ [9 Permission Handler modals]
                      │   ├─ ThemedToastContainer
                      │   ├─ GroupPermissionHandler
                      │   ├─ UpdateNotificationWrapper
                      │   └─ Switch (routes)
```

### Persistent UI elements

| Element | File | Description | Classification |
|---------|------|-------------|----------------|
| Dashboard shell | `src/lib/pages/Dashboard/index.tsx` | Sidebar + content layout, uses `makeStyles` from `navigation/style.ts` (partially dead refs) | CONTAINER |
| Menu (sidebar) | `src/lib/navigation/Menu.tsx` | Persistent left Drawer (320px width), contains business logic: profile CRUD, fund claiming, navigation, logout | CONTAINER — not just presentation |
| Profile | `src/lib/components/Profile.tsx` | Sidebar balance display, paginated `listOutputs`, listens to `balance-changed` event | BEHAVIORAL |
| Permission handlers | 9 components rendered at root level | Modal-based queue processing; render regardless of route | BEHAVIORAL |
| ThemedToastContainer | `src/lib/components/ThemedToastContainer.tsx` | react-toastify wrapper with theme-aware styling | VISUAL |
| AuthRedirector | `src/lib/navigation/AuthRedirector.tsx` | Invisible component that redirects based on auth state | CONTAINER |
| UpdateNotification | `src/lib/components/UpdateNotification.tsx` | Auto-update notification banner | VISUAL |

### Responsive breakpoints

Source: `UserInterface.tsx` lines 28–33

```typescript
const queries = {
  xs: '(max-width: 500px)',
  sm: '(max-width: 720px)',
  md: '(max-width: 1024px)',
  or: '(orientation: portrait)'
}
```

Used via `useBreakpoint()` hook from `src/lib/utils/useBreakpoints.tsx`.

> **Design note:** Sidebar balance display is flagged for compression to a header strip in the redesign. Currently lives in the 320px sidebar which is excessive width on smaller screens.

---

## 5. Patterns in Current Use

| Pattern | Current Implementation | Canonical File | Target |
|---------|----------------------|----------------|--------|
| Forms | Controlled inputs with `useState` | `src/lib/pages/Greeter/index.tsx` | Same pattern, shadcn Input/Select |
| Validation | Inline `error`/`helperText` on MUI TextField | `src/lib/pages/Dashboard/Payments/index.tsx` | Same pattern, shadcn form fields |
| Dialogs | `CustomDialog` wrapper or raw MUI `Dialog` | `src/lib/components/CustomDialog/index.tsx` | shadcn Dialog |
| Toasts | react-toastify `toast.success()` / `toast.error()` | Used everywhere | sonner `toast()` |
| Loading (inline) | MUI `CircularProgress` | Various | lucide `Loader2` with spin |
| Loading (page) | MUI `LinearProgress` or custom `PageLoading` | `src/lib/components/PageLoading/index.tsx` | shadcn Skeleton |
| Empty states | Centered subdued `Typography` | `src/lib/pages/Dashboard/Apps/index.tsx` | Designed empty states with illustrations |
| Error states | `ErrorBoundary` class component + try/catch + toast | `src/lib/components/ErrorBoundary.tsx` | Same pattern, updated visuals |
| Destructive confirm | MUI Dialog with explicit cancel/confirm buttons | `src/lib/navigation/Menu.tsx` (delete profile) | shadcn AlertDialog |
| Icons | `@mui/icons-material` imports | Used everywhere | lucide-react equivalents |
| Accordions | MUI `Accordion`/`AccordionSummary`/`AccordionDetails` | `src/lib/pages/Recovery/RecoverPassword.tsx` | shadcn Collapsible or Accordion |
| Tabs | MUI `Tabs`/`Tab` | `src/lib/pages/Dashboard/Payments/index.tsx` | shadcn Tabs |
| Tooltips | MUI `Tooltip` | Various | shadcn Tooltip |
| Copy to clipboard | `navigator.clipboard.writeText()` + toast | `src/lib/pages/Dashboard/MyIdentity/index.tsx` | Same, with shadcn toast |
| Theme toggle | 3-mode (light/dark/system) in localStorage | `src/lib/pages/Dashboard/Settings/index.tsx` | Same logic, shadcn Toggle/Select |

---

## 6. Internationalization

### Setup

| Aspect | Detail |
|--------|--------|
| Library | i18next v26.0.5 + react-i18next v17.0.3 |
| Config file | `src/lib/i18n/index.ts` |
| Options | `useSuspense: false`, `escapeValue: false`, fallback `'en'` |
| Language context | `src/lib/i18n/LanguageContext.tsx` — persists to localStorage key `bsv-desktop-language` |
| Detection | localStorage → `navigator.language` → `'en'` fallback |
| Translations file | `src/lib/i18n/translations.ts` — single file, ~11,400 lines, JS object export |

### Supported languages (12)

`en`, `es`, `zh`, `hi`, `fr`, `ar`, `pt`, `bn`, `ru`, `id`, `ja`, `pl`

### Usage pattern

```typescript
const { t } = useTranslation();
// ...
<Typography>{t('settings_title')}</Typography>
```

### Key conventions

- Format: `snake_case` with category prefix (e.g., `settings_title`, `payments_send`, `greeter_welcome`)
- Interpolation: `{{variable}}` syntax (e.g., `t('balance_display', { amount: 100 })`)
- All user-facing strings must use `t()` — no hardcoded strings

### Migration rule

No hardcoded user-facing strings. Reuse existing translation keys. New keys follow the `category_descriptor` convention.
