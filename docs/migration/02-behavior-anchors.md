# 02 — Behavior Anchors

> **Audience:** Claude Code (Step 3)
> **Purpose:** Verification checklist. Every route gets a behavior-anchor record. A migrated screen passes when all anchors hold.
>
> **Priority:** HIGHEST. This document is the source of truth for migration correctness.

---

## Table of Contents

### Top-Level Routes
1. [`/` — Greeter](#route-)
2. [`/recovery` — Recovery](#route-recovery)
3. [`/recovery/presentation-key` — RecoverPresentationKey](#route-recoverypresentation-key)
4. [`/recovery/password` — RecoverPassword](#route-recoverypassword)
5. [`/privacy` — PrivacyPolicy](#route-privacy)
6. [`/usage` — UsagePolicy](#route-usage)

### Dashboard Routes
7. [`/dashboard` — Dashboard Shell](#route-dashboard)
8. [`/dashboard/app-catalog` — AppCatalog](#route-dashboardapp-catalog)
9. [`/dashboard/apps` — Apps](#route-dashboardapps)
10. [`/dashboard/app` — App](#route-dashboardapp)
11. [`/dashboard/payments` — Payments](#route-dashboardpayments)
12. [`/dashboard/identity` — MyIdentity](#route-dashboardidentity)
13. [`/dashboard/trust` — Trust](#route-dashboardtrust)
14. [`/dashboard/security` — Security](#route-dashboardsecurity)
15. [`/dashboard/settings` — Settings](#route-dashboardsettings)
16. [`/dashboard/legacybridge` — LegacyBridge](#route-dashboardlegacybridge)
17. [`/dashboard/manage-app/:originator` — AppAccess](#route-dashboardmanage-apporiginator)
18. [`/dashboard/basket/:basketId` — BasketAccess](#route-dashboardbasketbasketid)
19. [`/dashboard/protocol/:protocolId/:securityLevel` — ProtocolAccess](#route-dashboardprotocolprotocolidsecuritylevel)
20. [`/dashboard/counterparty/:counterparty` — CounterpartyAccess](#route-dashboardcounterpartycounterparty)
21. [`/dashboard/certificate/:certType` — CertificateAccess](#route-dashboardcertificatecerttype)

### Redirects
22. [`/dashboard/counterparty/self` → `/dashboard/counterparty/{myIdentityKey}`](#redirects-1)
23. [`/dashboard/counterparty/anyone` → `/dashboard/counterparty/0279be...`](#redirects-1)

### Permission Handler Modals
24. [BasketAccessHandler](#basketaccesshandler)
25. [CertificateAccessHandler](#certificateaccesshandler)
26. [SpendingAuthorizationHandler](#spendingauthorizationhandler)
27. [ProtocolPermissionHandler](#protocolpermissionhandler)
28. [GroupPermissionHandler](#grouppermissionhandler)
29. [PasswordHandler](#passwordhandler)
30. [RecoveryKeyHandler](#recoverykeyhandler)
31. [FundingHandler](#fundinghandler)

### Ambiguous Items
32. [AMBIGUOUS: Dead Code and Undefined References](#ambiguous-items-1)

---

## Top-Level Routes

---

### Route: `/`

**Component file:** `src/lib/pages/Greeter/index.tsx`

**Data consumed:**
- `WalletContext`: `managers`, `configStatus`, `useWab`, `loginType`, `saveEnhancedSnapshot`, `initializingBackendServices`, `finalizeConfig`
- `UserContext`: `appVersion`, `appName`, `pageLoaded`
- `useTheme()` for MUI theme
- `useTranslation()` for i18n
- Local state: `entryMode` ('choose'|'create'|'login'), `step`, `phone`, `code`, `mnemonic`, `password`, `confirmPassword`, `accountStatus`, `loading`, `showPassword`, `showMnemonicDialog`, `mnemonicLocked`, `directKeyInput`, `directKeyLocked`, `showConfig`
- Refs: `phoneFieldRef`, `codeFieldRef`, `mnemonicFieldRef`, `passwordFieldRef`

**Actions emitted:**
- `finalizeConfig(wabConfig)` — sets network, storage, loginType globally
- `walletManager.startAuth({ phoneNumber })` — WAB phone auth initiation
- `walletManager.completeAuth({ phoneNumber, otp })` — OTP verification
- `walletManager.providePresentationKey(keyBytes)` — mnemonic-derived key via HD path m/0'/0/0
- `walletManager.providePrimaryKey(keyBytes)` — direct key submission
- `walletManager.providePrivilegedKeyManager(disabledManager)` — direct-key mode (no privileged ops)
- `walletManager.providePassword(password)` — final auth step
- `saveEnhancedSnapshot()` → `localStorage.snap` — persists wallet state
- `persistKeyMaterial(keyHex, mnemonic)` → localStorage — stores key material for Security page retrieval
- `navigator.clipboard.writeText()` — copy mnemonic/key
- `window.electronAPI.saveMnemonic()` / `savePrivateKey()` — IPC file save
- `history.push('/dashboard/apps')` — on successful auth
- Toast: success/error on every async step

**Conditional behavior:**
- `!pageLoaded` → renders `<PageLoading />`
- `initializingBackendServices === true` → header only, no forms
- `entryMode === 'choose'` → Create/Login buttons + app header
- `entryMode === 'create'|'login'` → Back + config toggle + stepper (when `!showConfig && configStatus === 'configured'`)
- `showConfig === true` → WalletConfig panel replaces stepper
- Three stepper configurations based on `loginType`:
  - `'wab'`: phone(0) → code(1) → password(2)
  - `'direct-key'`: directkey(0) only
  - `'mnemonic-advanced'`: presentation(0) → password(1)
- Password confirm field: shown only when `accountStatus === 'new-user'`
- All buttons disabled when `loading === true`
- Phone submit disabled if `!phone || phone.length < 10`
- Code submit disabled if `code.length !== 6`
- Mnemonic field locked after generation (`mnemonicLocked`)
- Recovery link disabled if `configStatus !== 'configured'`

**Edge cases:**
- `loading` state mutex prevents double-submit across all forms
- `directKeyInput`/`directKeyLocked` lifted to parent to survive re-renders (inline component state loss fix)
- Mnemonic generation locks field via `setMnemonicLocked(true)` — prevents overwrite
- `handleResendCode`: 2-second artificial delay after setLoading(false) to prevent double-SMS
- No debounce on text input onChange (immediate state updates, no side effects)
- Config changes mid-flow: `useEffect` with `[loginType]` dependency resets step
- Snapshot saved AFTER auth success, BEFORE navigation — critical ordering

**Routes in / routes out:**
- In: `/` (app loads unauthenticated, or AuthRedirector doesn't redirect)
- Out: `/dashboard/apps` (successful auth), `/recovery` (link), `/privacy` (link), `/usage` (link)

**Frozen vs flexible:**
- **Frozen:** Three auth modes, step progression, config finalization before auth, HD derivation path m/0'/0/0, snapshot timing, wallet manager method signatures, error propagation via toast, loading mutex
- **Flexible:** All visual layout, icons, colors, fonts, spacing, stepper appearance, dialog styling, spinner size, field labels (all via i18n)

---

### Route: `/recovery`

**Component file:** `src/lib/pages/Recovery/index.tsx`

**Data consumed:**
- `history` prop (react-router v5)
- `useTranslation()` for i18n

**Actions emitted:**
- `history.push('/recovery/presentation-key')` — Presentation Key option
- `history.push('/recovery/password')` — Password recovery option
- `history.go(-1)` — back button

**Conditional behavior:**
- None. Static two-option menu. No loading, error, or empty states.

**Edge cases:**
- No authentication guard — renders regardless of auth state
- No null checks on history

**Routes in / routes out:**
- In: `/recovery` (from Greeter recovery link)
- Out: `/recovery/presentation-key`, `/recovery/password`, history back

**Frozen vs flexible:**
- **Frozen:** Two-option structure, navigation paths, translation key names
- **Flexible:** All visual styling, icon choices, typography variants

---

### Route: `/recovery/presentation-key`

**Component file:** `src/lib/pages/Recovery/RecoverPresentationKey.tsx`

**Data consumed:**
- `WalletContext`: `managers`, `saveEnhancedSnapshot`, `network`
- `useTranslation()` for i18n
- `history` prop
- Local state: `accordianView` ('recovery-key'|'password'), `password`, `recoveryKey`, `loading`, `authenticated`
- Wallet manager properties: `.authenticated`, `.authenticationFlow` (setter), `.authenticationMode` (setter)

**Actions emitted:**
- Sets `walletManager.authenticationFlow = 'existing-user'`
- Sets `walletManager.authenticationMode = 'recovery-key-and-password'`
- `walletManager.provideRecoveryKey(bytes)` — base64-decoded key bytes
- `walletManager.providePassword(password)` — password string
- `walletManager.destroy()` — on logout confirmation
- `localStorage.snap = saveEnhancedSnapshot()` — after password submission
- `history.push('/dashboard/apps')` — on success
- `window.confirm()` — logout confirmation
- Toast: success/error

**Conditional behavior:**
- If `walletManager.authenticated === true` → renders logout + back buttons instead of form
- `loading` → buttons replaced with `<CircularProgress />`
- Only one accordion expanded at a time
- CheckCircle icon on password accordion when past recovery-key step

**Edge cases:**
- No cancel token for in-flight requests on rapid submissions
- Recovery key base64 format not validated before submission
- `window.confirm()` is modal-blocking

**Routes in / routes out:**
- In: `/recovery/presentation-key`
- Out: `/dashboard/apps` (success), history back

**Frozen vs flexible:**
- **Frozen:** Two-step flow (recovery-key → password), `authenticationFlow = 'existing-user'`, `authenticationMode = 'recovery-key-and-password'`, base64 encoding, localStorage key `snap`, navigation path `/dashboard/apps`
- **Flexible:** Accordion styling, icon appearance, typography, input field styling, toast styling

---

### Route: `/recovery/password`

**Component file:** `src/lib/pages/Recovery/RecoverPassword.tsx`

**Data consumed:**
- `WalletContext`: `managers`, `saveEnhancedSnapshot`, `useWab`
- `useTranslation()` for i18n
- `history` prop
- Local state: `accordianView` ('auth-method'|'phone'|'code'|'mnemonic'|'recovery-key-final'|'new-password'), form fields, `loading`, `authenticated`

**Actions emitted:**
- Sets `authenticationFlow = 'existing-user'`, `authenticationMode = 'presentation-key-and-recovery-key'`
- WAB path: `startAuth({ phoneNumber })`, `completeAuth({ phoneNumber, otp })`
- Self-custody path: `providePresentationKey(bytes)` via HD derivation m/0'/0/0
- Both paths: `provideRecoveryKey(bytes)`, `changePassword(string)`
- `walletManager.destroy()` on logout
- `localStorage.snap = saveEnhancedSnapshot()` after recovery-key and password
- `history.push('/dashboard/apps')` on success
- Toast: success/error

**Conditional behavior:**
- Authenticated guard: alternate render if wallet already authenticated
- Auth method branching on `useWab`:
  - WAB: phone → OTP → recovery key → password (4+ accordions)
  - Self-custody: mnemonic → recovery key → password (3 accordions)
- Only one accordion expanded at a time
- Password mismatch validation: client-side check before API call
- Resend code: disabled during `loading`

**Edge cases:**
- HD derivation path hardcoded m/0'/0/0
- 2-second artificial delay after code resend (prevents UI flicker)
- No mnemonic format validation before HD derivation (fails at derivation point)
- No base64 format validation on recovery key
- No request cancellation on unmount

**Routes in / routes out:**
- In: `/recovery/password`
- Out: `/dashboard/apps` (success), history back

**Frozen vs flexible:**
- **Frozen:** Multi-stage flow, `useWab` branching, HD path m/0'/0/0, base64 encoding, 2-second resend delay, password confirmation requirement, auth mode constants
- **Flexible:** All accordion styling, input fields, typography, icons

---

### Route: `/privacy`

**Component file:** `src/lib/pages/Policies/privacy.tsx`

**Data consumed:**
- None. Static HTML/JSX content.
- CSS: `./styles.css`

**Actions emitted:**
- `history.back()` via `<a href="javascript:history.back()">`

**Conditional behavior:**
- None. Static content.

**Edge cases:**
- Uses deprecated `javascript:` protocol in href
- Not using i18n — hardcoded English

**Routes in / routes out:**
- In: `/privacy` (from Greeter footer)
- Out: history back

**Frozen vs flexible:**
- **Frozen:** Policy text content, section structure, contact emails
- **Flexible:** All CSS styling

---

### Route: `/usage`

**Component file:** `src/lib/pages/Policies/usage.tsx`

**Data consumed:**
- None. Static HTML/JSX content.
- CSS: `./styles.css`

**Actions emitted:**
- `history.back()` via `<a href="javascript:history.back()">`

**Conditional behavior:**
- None. Static content.

**Edge cases:**
- Same as `/privacy` — deprecated `javascript:` protocol, no i18n

**Routes in / routes out:**
- In: `/usage` (from Greeter footer)
- Out: history back

**Frozen vs flexible:**
- **Frozen:** Terms text content, section structure
- **Flexible:** All CSS styling

---

## Dashboard Routes

---

### Route: `/dashboard`

**Component file:** `src/lib/pages/Dashboard/index.tsx`

**Data consumed:**
- `UserContext`: `pageLoaded`
- `WalletContext`: `activeProfile`, `managers`, `adminOriginator`
- `useHistory()`, `useBreakpoint()`
- `makeStyles(style)` from `navigation/style.ts` (partially dead refs — see AMBIGUOUS section)
- Local state: `menuOpen` (default true), `myIdentityKey` (default 'self')

**Actions emitted:**
- Balance check on mount: `permissionsManager.listOutputs({ basket: 'default', limit: 1 })` → redirects to `/dashboard/legacybridge` if `totalOutputs === 0`
- Menu toggle via `setMenuOpen`

**Conditional behavior:**
- `!pageLoaded` → renders `<PageLoading />`
- Sidebar visible when `menuOpen && !breakpoints.sm`
- Mobile hamburger: shown only when `breakpoints.sm`
- Content margin: `320px` when sidebar open (desktop), `0px` when closed or mobile
- ErrorBoundary wraps all child routes

**Edge cases:**
- Balance check runs once on mount (empty deps array) — no re-check on profile switch
- `myIdentityKey` hardcoded to 'self' (TODO comment in source)
- `profileKey` derived from `activeProfile?.id ?? activeProfile?.name ?? 'none'` — forces re-render on profile switch

**Routes in / routes out:**
- In: `/dashboard` (from Greeter success, from AuthRedirector)
- Out: Child routes via `<Switch>` (see table of contents)

**Frozen vs flexible:**
- **Frozen:** Zero-balance redirect to legacybridge, sidebar toggle logic, ErrorBoundary wrap, child route structure
- **Flexible:** Sidebar width (320px), content margin transition, hamburger icon, all styling

**Redirects:**
- `/dashboard/counterparty/self` → `/dashboard/counterparty/{myIdentityKey}`
- `/dashboard/counterparty/anyone` → `/dashboard/counterparty/0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`

---

### Route: `/dashboard/app-catalog`

**Component file:** `src/lib/pages/Dashboard/AppCatalog/index.tsx`

**Data consumed:**
- No context dependencies (standalone)
- External API: `AppCatalogAPI().findApps()` from `metanet-apps`
- Fuse.js for search (threshold 0.3, keys: name/description/tags/category)
- Local state: `catalogLoading`, `currentView` ('list'|'details'), `selectedApp`, `search`, `openModal`, `activeScreenshot`, `isExpanded`

**Actions emitted:**
- Real-time Fuse.js filtering on search input
- `window.open(httpURL || 'https://' + domain)` — open app in browser
- Modal open/close for screenshot enlargement
- `history.push('/dashboard/apps')` — back navigation

**Conditional behavior:**
- `catalogLoading` → rotating AppLogo spinner
- `filteredCatalogApps.length === 0` after load → "no apps found" message
- `currentView === 'list'` → grid of app cards with search
- `currentView === 'details'` → single app detail with banner, description, metadata, screenshots
- Tags: show first 2 + "+N more" count if >2
- Screenshots: horizontal carousel with dot indicators
- Search input: expands to 70% on focus, contracts to 20em on blur (0.3s transition)

**Edge cases:**
- Icon fallback to metanetapps.com/favicon.ico on load failure
- No banner: skip section. No tags: skip section. No screenshots: skip section. No changelog: skip section.
- Search is fuzzy (Fuse.js threshold 0.3)

**Routes in / routes out:**
- In: `/dashboard/app-catalog` (from Menu, from AuthRedirector)
- Out: `/dashboard/apps` (back button), external URLs (app launch)

**Frozen vs flexible:**
- **Frozen:** Two-view architecture (list/details), search threshold and keys, grid layout (xs:12 sm:6 md:4), tag limit (show 2 + count), Fuse.js dependency
- **Flexible:** Card hover effects, search input sizing, carousel styling, typography, colors

---

### Route: `/dashboard/apps`

**Component file:** `src/lib/pages/Dashboard/Apps/index.tsx`

**Data consumed:**
- `WalletContext`: `managers.permissionsManager`, `adminOriginator`, `network`
- Translation keys: `apps_*`
- Local state: `actions`, `loading`, `hasMore`, `loadingMore`, `copyingTxid`
- SDK: `WalletAction[]`

**Actions emitted:**
- `permissionsManager.listActions()` with pagination (PAGE_SIZE = 30)
- `navigator.clipboard.writeText(txid)` — copy TXID
- `window.open()` — WhatsOnChain explorer link (different URLs for mainnet/testnet)
- Toast: success/error on copy

**Conditional behavior:**
- `loading` → centered CircularProgress, min height 300px
- `actions.length === 0` after load → empty state message
- `hasMore` → "Load More" button visible
- `loadingMore` → "Load More" disabled, shows "Loading..."
- Explorer/copy buttons hidden for ABORTABLE_STATUSES: `['unsigned', 'nosend', 'nonfinal']`
- Status colors: green (completed/unproven/sending), warning (nosend/unsigned/nonfinal), error (failed)

**Edge cases:**
- `cancelled` flag in useEffect cleanup prevents stale state updates
- Pagination: probe page 0 first to get totalActions, then jump to last page (walks backward)
- TXID display: index as fallback key if txid missing

**Routes in / routes out:**
- In: `/dashboard/apps` (from Greeter success, from Menu)
- Out: External WhatsOnChain URLs (new tab). No route navigation.

**Frozen vs flexible:**
- **Frozen:** PAGE_SIZE (30), ABORTABLE_STATUSES set, status color mapping, explorer URL logic (mainnet vs testnet), backward pagination strategy
- **Flexible:** Typography, spacing, icon choices, toast messages

---

### Route: `/dashboard/app`

**Component file:** `src/lib/pages/Dashboard/App/Index.tsx`

**Data consumed:**
- `WalletContext`: `managers.permissionsManager`, `adminOriginator`, `network`
- Router location state: `domain`, `appName`, `iconImageUrl`
- sessionStorage: `lastAppDomain`, `lastAppName`, `lastAppIcon`
- localStorage: `transactions_{appDomain}` (first-page cache)
- LRU cache: 25 app pages in memory
- Local state: `isFetching`, `allActionsShown`, `copied`

**Actions emitted:**
- `permissionsManager.listActions()` with pagination (LIMIT = 10)
- `navigator.clipboard.writeText(url)` — copy app URL
- `window.open(url, '_blank')` — open app
- AbortController cancellation on unmount/page change

**Conditional behavior:**
- `isFetching` → disable "Load More", show loading state
- `allActionsShown` → hide "Load More"
- `copied` → CheckIcon for 2s, then revert

**Edge cases:**
- Dual caching: localStorage for page 0 only, in-memory LRU for up to 25 apps
- sessionStorage persistence for cross-session app state
- AbortController cancels pending requests on unmount
- Copy button debounce: 2-second reset
- Intentionally excludes `fetchPage` from useEffect deps to avoid double-fetches

**Routes in / routes out:**
- In: Pushed from Apps list with state `{ domain, appName, iconImageUrl }`
- Out: Back via PageHeader, external app URL (new tab)

**Frozen vs flexible:**
- **Frozen:** LIMIT (10), LRU cache capacity (25), copy timeout (2s), abort signal pattern, dual caching strategy
- **Flexible:** sessionStorage key names, typography, spacing, icon choices

---

### Route: `/dashboard/payments`

**Component file:** `src/lib/pages/Dashboard/Payments/index.tsx`

**Data consumed:**
- `WalletContext`: `managers`, `activeProfile`, `peerPayClient`, `loginType`, `adminOriginator`, `useMessageBox`, `messageBoxUrl`, `isHostAnointed`, `anointCurrentHost`
- `wallet.listProfiles()`, `wallet.listActions()`
- `PeerPayClient`: `sendPayment()`, `acceptPayment()`, `listIncomingPayments()`, `listIncomingPaymentRequests()`
- `CurrencyConverter`: `initialize()`, `convertToSatoshis()`, `getCurrencySymbol()`
- localStorage: `payReq_minAmount_{idSuffix}`, `payReq_maxAmount_{idSuffix}`

**Actions emitted:**
- `peerPayClient.sendPayment()` with recipient/amount
- `peerPayClient.acceptPayment()` with retry on failure (refetch + retry fresh copy)
- `wallet.listActions()` with 'peerpay' label
- `window.dispatchEvent(new CustomEvent('balance-changed'))` on accept success
- Toast: success/error

**Conditional behavior:**
- `!useMessageBox || !messageBoxUrl || !peerPayClient` → show `<MessageBoxConfig>` instead of main UI
- `!isHostAnointed` → blue info Alert
- `loginType === 'direct-key'` → hide internal transfer tab, no profiles list
- 4 tabs: Send Payment (0), Request Payment (1), Incoming Requests (2), Pending Payments (3)
- Tabs 2/3: lazy-load on switch
- Send button: disabled if `!recipient || amount <= 0 || sending`
- `sending` → "Sending..." text
- Per-message loading state: prevents button collision on multiple payments

**Edge cases:**
- Payment accept retry: on failure → refetch payment list → retry with fresh copy
- Profile filtering: exclude current active profile from internal transfer dropdown
- Currency conversion: async initialization, converts display currency to satoshis
- Manual public key validation: try `PublicKey.fromString()`, catch on invalid
- Identity search: deduplication by identityKey
- Min/max amount filters in localStorage (defaults: 1000–10M satoshis)

**Routes in / routes out:**
- In: `/dashboard/payments` (from Menu)
- Out: None (embedded components only)

**Frozen vs flexible:**
- **Frozen:** Four-tab structure, public key validation, retry strategy, balance-changed event name, localStorage key format, payment label 'peerpay', incoming payment limit (100)
- **Flexible:** Tab labels, icon choices, currency display, profile selection UI, alert styling

---

### Route: `/dashboard/identity`

**Component file:** `src/lib/pages/Dashboard/MyIdentity/index.tsx`

**Data consumed:**
- `WalletContext`: `managers.permissionsManager`, `network`, `adminOriginator`, `activeProfile`, `loginType`
- `ProtoWallet`, `VerifiableCertificate` from SDK
- localStorage: `provenCertificates_{profileId}` (cache)
- Wallet: `listCertificates({ certifiers: [], types: [], limit: 100 })`, `proveCertificate()`, `getPublicKey()`

**Actions emitted:**
- `permissionsManager.getPublicKey({ identityKey: true })` — primary key
- `permissionsManager.getPublicKey({ identityKey: true, privileged: true, privilegedReason })` — privileged key reveal
- `proveCertificate()` for each certificate → decrypt fields with `ProtoWallet('anyone')`
- `navigator.clipboard.writeText()` — copy keys
- Certificate revocation → remove from state + update localStorage cache

**Conditional behavior:**
- `primaryIdentityKey === '...'` → placeholder while fetching
- `privilegedIdentityKey === '...'` → "Reveal Key" button
- Privileged key revealed → full key with copy button
- `loginType === 'direct-key'` → hide privileged key section entirely
- `!activeProfile` → early return in useEffect (skip load)
- Certificates cached → load from cache first
- Certificate prove fails → log and skip
- Copy: CheckIcon for 2s
- Search filter: currently disabled (always returns false)

**Edge cases:**
- Cache key per profile: `provenCertificates_{profileId}`
- Verifier key: hardcoded standard key `0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`
- Decryption with `ProtoWallet('anyone')` for public field disclosure
- Copy button 2-second debounce

**Routes in / routes out:**
- In: `/dashboard/identity` (from Menu)
- Out: None

**Frozen vs flexible:**
- **Frozen:** Verifier key, cache key format, certificate listing params (limit: 100), key reveal flow, ProtoWallet('anyone'), early return for direct-key + no profile
- **Flexible:** Typography, icon choices, network title variants, certificate card styling

---

### Route: `/dashboard/trust`

**Component file:** `src/lib/pages/Dashboard/Trust/index.tsx`

**Data consumed:**
- `WalletContext`: `settings.trustSettings.{ trustLevel, trustedCertifiers }`, `updateSettings()`
- `useHistory()` for navigation blocking
- Custom utility: `arraysOfObjectsAreEqual()`

**Actions emitted:**
- `updateSettings()` with new trust settings
- Navigation blocking: `history.block()` returns false if unsaved changes
- Toast: save progress, success, error

**Conditional behavior:**
- `settingsLoading` → LinearProgress, disabled buttons
- `settingsNeedsUpdate` → snackbar "Save" button + Prompt on leave + NavigationConfirmModal
- `trustLevel > totalTrustPoints` → auto-cap to total
- Search filter: case-insensitive on name/description
- Slider disabled when `totalTrustPoints === 0`
- Navigation confirm: save-and-navigate or discard-and-navigate

**Edge cases:**
- Dirty detection: custom array equality check
- Navigation blocking: `history.block()` + custom modal + browser Prompt
- Auto-cap on entity removal if total drops below current level
- Deep clone: `JSON.parse(JSON.stringify())` for settings
- Discard navigation: 100ms setTimeout before navigate (ensures state update completes)

**Routes in / routes out:**
- In: `/dashboard/trust` (from Menu)
- Out: Blocked on unsaved changes → modal → requested pathname after save/discard

**Frozen vs flexible:**
- **Frozen:** Dirty detection logic, navigation blocking pattern, slider range (1 to totalTrustPoints), auto-cap, deep clone strategy
- **Flexible:** Entity card styling, modal styling, slider appearance, typography

---

### Route: `/dashboard/security`

**Component file:** `src/lib/pages/Dashboard/Security/index.tsx`

**Data consumed:**
- `UserContext`: `pageLoaded`
- `WalletContext`: `loginType`
- `useExportDataToFile()` hook
- `reconcileStoredKeyMaterial()` → `{ keyHex, mnemonic }`
- `useTranslation()`

**Actions emitted:**
- `navigator.clipboard.writeText()` — copy secrets
- `exportData()` with text/plain MIME type — file download
- Toast: success/error on download
- Dialog state: show/hide recovery key

**Conditional behavior:**
- `loginType === 'direct-key'` → reveal buttons for mnemonic/hex (if keys exist)
- Recovery mode → ChangePassword + RecoveryKey child components
- "Reveal Now" button only when `!showSecrets`; secrets render when `showSecrets === true`
- Copy disabled while `copied === true` (2s timeout)
- Word count display only if `hasMnemonic && phraseWordCount > 0`
- No keys found → message "No keys found"

**Edge cases:**
- Copy state timeout: 2000ms
- Mnemonic display: split by whitespace, numbered `${idx + 1}. ${word}`
- Private key hex: monospace, word-break: break-all

**Routes in / routes out:**
- In: `/dashboard/security` (from Menu)
- Out: None (dialog-only)

**Frozen vs flexible:**
- **Frozen:** Two-view architecture (direct-key vs recovery), reveal confirmation workflow, copy/paste with feedback
- **Flexible:** Typography, dialog styling, icon choices, alert severity

---

### Route: `/dashboard/settings`

**Component file:** `src/lib/pages/Dashboard/Settings/index.tsx`

**Data consumed:**
- `WalletContext`: `settings`, `updateSettings()`, `wabUrl`, `useRemoteStorage`, `useMessageBox`, `storageUrl`, `useWab`, `messageBoxUrl`, `backupStorageUrls`, `addBackupStorageUrl()`, `removeBackupStorageUrl()`, `syncBackupStorage()`, `permissionsConfig`, `updatePermissionsConfig()`
- `UserContext`: `pageLoaded`, `setManualUpdateInfo()`
- `useLanguage()`: `currentLanguage`, `setCurrentLanguage()`, `supportedLanguages`
- `useTheme()`
- `window.electronAPI.updates.check()` for update checks

**Actions emitted:**
- `updateSettings()` — theme/currency changes
- Backup storage: `addBackupStorageUrl()`, `removeBackupStorageUrl()`, `syncBackupStorage(progressCallback)`
- `updatePermissionsConfig()` → `window.location.reload()`
- `setCurrentLanguage(lang)` — no reload
- `window.electronAPI.updates.check()` → `setManualUpdateInfo()`
- Toast: success/error

**Conditional behavior:**
- `settingsLoading` → LinearProgress, disabled buttons
- Currency/theme buttons: disabled during loading; visual selection state
- Backup "Add Local Storage": only if `useRemoteStorage && !backupStorageUrls.includes('LOCAL_STORAGE')`
- Sync button: disabled if `syncLoading || backupLoading`
- Sync progress dialog: cannot close while `syncLoading`; shows error alert if `syncError`
- Permissions section: collapsible toggle
- Permissions save: reloads entire page

**Edge cases:**
- Theme/currency idempotence: early return if already selected
- Sync progress: logs split by newlines, accumulated in array (state batching)
- Backup URL validation: empty prevented by button disable + toast
- Permissions reset: resets local state only (no network call)
- Update check dev fallback: hardcoded sample data if GitHub API fails

**Routes in / routes out:**
- In: `/dashboard/settings` (from Menu)
- Out: None. `window.location.reload()` on permissions save.

**Frozen vs flexible:**
- **Frozen:** Currency/theme selectors as button grids, backup storage list, sync progress log (monospace, scrollable), permissions with expandable subsections, "At a Glance" section, page reload on permissions save
- **Flexible:** Button sizing, dialog styling, typography, icon choices, progress bar appearance, chip styling

---

### Route: `/dashboard/legacybridge`

**Component file:** `src/lib/pages/Dashboard/LegacyBridge/index.tsx`

**Data consumed:**
- `WalletContext`: `managers` (permissionsManager), `network`, `adminOriginator`
- Wallet methods: `getPublicKey()` (BRC-29), `listOutputs()`, `listActions()`, `internalizeAction()`, `createAction()`
- External API: WhatsOnChain `fetch('https://api.whatsonchain.com/v1/bsv/.../unspent/all')` (rate-limited)
- `getBeefForTxid()` utility
- `Utils.toBase64()`, `Utils.toArray()`

**Actions emitted:**
- Address derivation: `getPublicKey()` → PublicKey → Address (date-based prefix)
- UTXO import: `internalizeAction()` with BEEF + outputs + labels
- Transaction creation: `createAction()` with P2PKH locking script
- Balance polling: 3s interval checking all days' balances
- Auto-import trigger on positive balance
- `navigator.clipboard.writeText()` — copy address
- `window.dispatchEvent(new CustomEvent('balance-changed'))`
- Toast: success/error on import, send

**Conditional behavior:**
- Two tabs: Receive (0) / Send (1)
- `isLoadingAddress` → spinner while deriving address
- `balance === -1` → "Checking..." + spinner
- Send button disabled: `isSending || !recipientAddress || (!sweepMax && !amount)`
- Amount field disabled when `sweepMax === true`; MAX switch sets amount to max value
- Date navigation: back arrow disabled at day 0, forward disabled at today
- Processed transactions panel: only on tab 0 with data
- Auto-import flag: `isImportingRef` (useRef) prevents concurrent imports

**Edge cases:**
- Derivation prefix: Base64 encoded date string (YYYY-MM-DD)
- UTXO deduplication via `internalizedCacheRef`
- BEEF merging for multiple UTXOs
- Network validation: mainnet address must start with '1'
- Polling race condition guard: checks `isImportingRef.current` before fetch
- Timestamp labels: `ts:${Date.now()}` for tracking

**Routes in / routes out:**
- In: `/dashboard/legacybridge` (from Menu, from zero-balance redirect)
- Out: None (custom event for parent balance refresh)

**Frozen vs flexible:**
- **Frozen:** Two-tab architecture, date-based address derivation, BEEF import workflow, P2PKH locking, auto-import on positive balance, processed tx list, 3s polling interval
- **Flexible:** Tab styling, QR code sizing, card layouts, alert styling, typography

---

### Route: `/dashboard/manage-app/:originator`

**Component file:** `src/lib/pages/Dashboard/AppAccess/index.tsx`

**Data consumed:**
- Route param: `originator` (URL-encoded)
- `WalletContext.managers`
- `useHistory()` for tab persistence via `history.appAccessTab`
- `DEFAULT_APP_ICON` constant

**Actions emitted:**
- `window.open(url, '_blank')` — app launch (adds `https://` if needed)
- `navigator.clipboard.writeText(url)` — copy URL
- Tab change + store in `history.appAccessTab`
- Toast: error on app data fetch

**Conditional behavior:**
- `loading` → full-height centered CircularProgress
- `error` → error message Typography
- App data fallback: placeholder on error
- Four tabs: Protocols (0), Spending (1), Baskets (2), Certificates (3)
- Copy: disabled during 2s feedback

**Edge cases:**
- Domain parsing: first part before dot, capitalized
- Originator URL-decoded on mount
- Tab cleanup on unmount (clears `history.appAccessTab`)

**Routes in / routes out:**
- In: `/dashboard/manage-app/:originator`
- Out: External app launch (new tab). PageHeader back button.

**Frozen vs flexible:**
- **Frozen:** Four-tab architecture, PageHeader with back/launch/copy, description text, permission list components
- **Flexible:** Tab styling, typography, icon choices, button variants

---

### Route: `/dashboard/basket/:basketId`

**Component file:** `src/lib/pages/Dashboard/BasketAccess/index.tsx`

**Data consumed:**
- Route param: `basketId`
- Route state: `{ id, name, description, iconURL, documentationURL }` (optional fast path)
- `WalletContext`: `managers`, `adminOriginator`, `settings.trustSettings.trustedCertifiers`
- `UserContext.onDownloadFile()`
- Wallet: `listOutputs()`, `listActions()`
- `RegistryClient` for metadata resolution

**Actions emitted:**
- `navigator.clipboard.writeText()` — copy basket ID
- `onDownloadFile(blob, filename)` — export JSON of outputs
- Revoke all access: `window.confirm()` (implementation stubbed)
- Toast: error on fetch/export

**Conditional behavior:**
- `loading` → spinner
- Route state provided → skip RegistryClient fetch (fast path)
- Trust scoring: select highest-trust result from registry
- Documentation link: only if `documentationURL` truthy
- Items count display

**Edge cases:**
- Trust scoring: max trust value, first match on tie
- Basket ID truncation: first 6 chars in fallback
- Revoke: `window.confirm()` only, actual revocation commented out
- Empty outputs: sets empty array, no error

**Routes in / routes out:**
- In: `/dashboard/basket/:basketId`
- Out: File download. PageHeader back button.

**Frozen vs flexible:**
- **Frozen:** PageHeader with back/export, basket description + docs link, apps-with-access list, revoke button, trust-based registry lookup
- **Flexible:** Typography, card styling, button colors

---

### Route: `/dashboard/protocol/:protocolId/:securityLevel`

**Component file:** `src/lib/pages/Dashboard/ProtocolAccess/index.tsx`

**Data consumed:**
- Route params: `protocolId` (URL-encoded), `securityLevel` (URL-encoded number)
- Route state: `{ protocolName, iconURL, description, documentationURL }` (optional fast path)
- `WalletContext`: `managers`, `settings.trustSettings.trustedCertifiers`, `adminOriginator`
- `RegistryClient` for metadata

**Actions emitted:**
- `navigator.clipboard.writeText(protocolID)` — copy ID
- Toast: error on fetch

**Conditional behavior:**
- `loading` → spinner (initially `!locationState`)
- Route state provided → skip fetch
- Trust scoring: highest-trust certifier result
- Documentation link: only if truthy
- Copy: 2s disabled

**Edge cases:**
- Protocol ID and security level both URL-decoded
- `Number(decodeURIComponent(encodedSecurityLevel))` for type conversion
- Fast path: early return if `protocolDetails && locationState`

**Routes in / routes out:**
- In: `/dashboard/protocol/:protocolId/:securityLevel`
- Out: PageHeader back button.

**Frozen vs flexible:**
- **Frozen:** PageHeader with back/copy, security level display, protocol description, docs link, apps-with-access list, trust-based lookup
- **Flexible:** Typography, card styling, bold weight

---

### Route: `/dashboard/counterparty/:counterparty`

**Component file:** `src/lib/pages/Dashboard/CounterpartyAccess/index.tsx`

**Data consumed:**
- Route param: `counterparty` (identity key)
- `WalletContext`: `managers`, `adminOriginator`
- `IdentityClient.resolveByIdentityKey()` → `DisplayableIdentity[]`

**Actions emitted:**
- `navigator.clipboard.writeText(counterparty)` — copy key
- Toast: error on identity/trust fetch

**Conditional behavior:**
- `loadingIdentity || loadingTrust` → spinner
- `error` → error Typography
- Identity fetch: first result = identity, rest = trust endorsements
- No results → placeholder with abbreviated key (first 6 chars)
- Three tabs: Trust Endorsements (CounterpartyChips), Protocol Access (ProtocolPermissionList), Certificates (placeholder)
- Empty endorsements → fallback message

**Edge cases:**
- Abbreviated key: first 6 chars + ellipsis
- Error accumulation with semicolon separator
- Icon display: only if not loading

**Routes in / routes out:**
- In: `/dashboard/counterparty/:counterparty`
- Out: PageHeader back. Navigation via chips/lists.

**Frozen vs flexible:**
- **Frozen:** Three-tab architecture, trust endorsements as chips, protocol access list, certificate placeholder, identity resolution via IdentityClient
- **Flexible:** Tab styling, chip appearance, typography

---

### Route: `/dashboard/certificate/:certType`

**Component file:** `src/lib/pages/Dashboard/CertificateAccess/index.tsx`

**Data consumed:**
- Route param: `certType` (URL-encoded)
- `WalletContext`: `managers`, `settings.trustSettings.trustedCertifiers`, `adminOriginator`
- `RegistryClient` for certificate definition
- `Img` from `@bsv/uhrp-react` for field icons

**Actions emitted:**
- `navigator.clipboard.writeText(certType)` — copy type
- Toast: error on fetch

**Conditional behavior:**
- `loading` → spinner
- Empty registry results → "Custom Certificate" placeholder (not yet registered)
- Trust scoring: highest-trust result
- Fields: list of `Object.entries(fields)` with optional field icons
- Documentation link: only if truthy
- Copy: 2s disabled

**Edge cases:**
- Unregistered certificate: placeholder message
- Field icon: `Img` with uhrp-react, 75% width/height
- Certificate type truncation: first 6 chars in fallback

**Routes in / routes out:**
- In: `/dashboard/certificate/:certType`
- Out: PageHeader back.

**Frozen vs flexible:**
- **Frozen:** PageHeader with back/copy, cert description, docs link, fields list with icons, trust-based lookup
- **Flexible:** Typography, avatar styling, field layout

---

## Redirects

| Source | Target | Mechanism |
|--------|--------|-----------|
| `/dashboard/counterparty/self` | `/dashboard/counterparty/{myIdentityKey}` | `<Redirect>` in Dashboard Switch |
| `/dashboard/counterparty/anyone` | `/dashboard/counterparty/0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798` | `<Redirect>` in Dashboard Switch |

Note: `myIdentityKey` is currently hardcoded to `'self'` in Dashboard (TODO comment).

---

## Permission Handler Modals

These are rendered at root level in `UserInterface.tsx`, not inside any route. They are queue-based and can appear over any page.

---

### BasketAccessHandler

**Component file:** `src/lib/components/BasketAccessHandler/index.tsx`

**Data consumed:**
- `WalletContext`: `basketRequests` (queue), `advanceBasketQueue`, `managers.permissionsManager`
- `UserContext`: `basketAccessModalOpen`

**Actions emitted:**
- `permissionsManager.denyPermission(requestID)` — deny
- `permissionsManager.grantPermission({ requestID })` — grant
- `advanceBasketQueue()` — advance to next request

**Conditional behavior:**
- Returns `null` if `!basketAccessModalOpen || !basketRequests.length`
- Reason section: conditional on `reason` field
- Renewal title: dynamic based on `renewal` boolean

**Edge cases:**
- Empty queue safety check before accessing `[0]`
- Both handlers synchronous (no loading states)
- Visual signature: deterministic color hash on entire request object

**Frozen vs flexible:**
- **Frozen:** Queue processing, two-button (deny/grant), request shape
- **Flexible:** Dialog styling, button labels (i18n)

---

### CertificateAccessHandler

**Component file:** `src/lib/components/CertificateAccessHandler/index.tsx`

**Data consumed:**
- `WalletContext`: `certificateRequests` (queue), `advanceCertificateQueue`, `managers.permissionsManager`
- `UserContext`: `certificateAccessModalOpen`
- Local state: `granting`, `denying`

**Actions emitted:**
- Async `denyPermission(requestID)` → dispatches `cert-access-changed` event `{ op: 'deny', originator }`
- Async `grantPermission({ requestID })` → dispatches `cert-access-changed` event `{ op: 'grant', originator }`
- `advanceCertificateQueue()` in finally block

**Conditional behavior:**
- Returns `null` if `!certificateAccessModalOpen || !certificateRequests.length`
- Description: conditional on `description` field
- Renewal title: dynamic
- Both buttons disabled if `granting || denying`
- Grant button: CircularProgress spinner during `granting`

**Edge cases:**
- Async with try/finally ensures cleanup
- Custom event dispatch even if async fails (in finally)

**Frozen vs flexible:**
- **Frozen:** Async pattern, custom event dispatch, queue advancement in finally, loading disabled state
- **Flexible:** Dialog styling, button labels

---

### SpendingAuthorizationHandler

**Component file:** `src/lib/components/SpendingAuthorizationHandler/index.tsx`

**Data consumed:**
- `WalletContext`: `managers.permissionsManager`, `spendingRequests` (queue), `advanceSpendingQueue`
- `UserContext`: `spendingAuthorizationModalOpen`
- Local state: `detailsOpen`

**Actions emitted:**
- Synchronous `denyPermission(requestID)` — deny
- `grantPermission({ requestID, ephemeral: singular, amount })` — grant with conditional amount
  - Normal: `singular: true`, no amount (one-time)
  - Limit increase/create: `singular: false`, specific amount (persistent)
- `advanceSpendingQueue()`

**Conditional behavior:**
- Returns `null` if `spendingRequests.length === 0`
- Three request type UIs detected via description string:
  - "Increase spending limit" → centered amount + monthly suffix
  - "Create a spending limit" → centered amount + info note
  - Normal → amount + collapsible line items
- Line items collapse: only if `lineItems?.length > 1`
- Button variants: green approve for limits, green authorize for normal
- Cancel vs Deny label differs by type

**Edge cases:**
- Amount undefined for normal requests; specific value for limits
- No loading state (synchronous)
- Description string matching for type detection

**Frozen vs flexible:**
- **Frozen:** Three-type detection (string matching), ephemeral flag logic, amount passing
- **Flexible:** Dialog titles, amounts, line item display, button labels

---

### ProtocolPermissionHandler

**Component file:** `src/lib/components/ProtocolPermissionHandler/index.tsx`

**Data consumed:**
- `WalletContext`: `protocolRequests` (queue), `advanceProtocolQueue`, `managers.permissionsManager`
- `UserContext`: `protocolAccessModalOpen`
- Local: `permissionTypeDocs` map (identity/renewal/basket/protocol)

**Actions emitted:**
- Async deny → dispatches `protocol-permissions-changed` event `{ op: 'deny', originator, protocolID, protocolSecurityLevel, counterparty }`
- Async grant → dispatches `protocol-permissions-changed` event `{ op: 'grant', ... }`
- Queue advance in `.finally()`

**Conditional behavior:**
- Returns `null` if `!protocolAccessModalOpen || !protocolRequests.length`
- Type-driven UI: `getPermissionTypeDoc()` determines title/description/icon based on `type || 'protocol'`
- Description: conditional on field
- Counterparty: conditional on field

**Edge cases:**
- Missing type defaults to 'protocol'
- Missing originator shows 'unknown'
- Promise-finally ensures event dispatch before queue advance

**Frozen vs flexible:**
- **Frozen:** Permission type doc map, promise-finally pattern, event dispatch, queue advancement
- **Flexible:** Type strings, dialog titles, icon types

---

### GroupPermissionHandler

**Component file:** `src/lib/components/GroupPermissionHandler/index.tsx`

**Data consumed:**
- `WalletContext`: `groupPermissionRequests` (queue), `advanceGroupQueue`, `managers.permissionsManager`
- `UserContext`: `groupPermissionModalOpen`
- Local state: `originator`, `requestID`, `spendingAuthorization`, `protocolPermissions`, `basketAccess`, `certificateAccess`, `isGranting`

**Actions emitted:**
- Async `denyGroupedPermission(requestID)` — deny all
- Async `grantGroupedPermission({ requestID, granted, expiry: 0 })` — grant filtered permissions
  - Builds `granted` object: only items where `enabled: true`, strips `enabled` flag
- `advanceGroupQueue()` in finally

**Conditional behavior:**
- `open={groupPermissionModalOpen && groupPermissionRequests.length > 0}`
- Each section renders only if its array length > 0 (spending, protocol, certificate, basket)
- Per-item toggle checkboxes: `toggleProtocolPermission`, `toggleCertificateAccess`, `toggleBasketAccess`
- Both buttons disabled if `isGranting`
- Grant button: "Granting..." with spinner during async

**Edge cases:**
- State resets on queue change (useEffect on `groupPermissionRequests`)
- Initial setup: maps items adding `{ enabled: true }`
- Certificate fields normalization: handles both array and object formats
- Errors logged but not surfaced (silent failures)
- `expiry: 0` hardcoded (TODO in source)

**Frozen vs flexible:**
- **Frozen:** Multi-state toggle, checkbox selection, enabled-filtering before grant, async grant/deny
- **Flexible:** Section rendering, checkbox labels, button text

---

### PasswordHandler

**Component file:** `src/lib/components/PasswordHandler.tsx`

**Data consumed:**
- `UserContext`: `onFocusRequested`, `onFocusRelinquished`, `isFocused`
- `WalletContext`: `setPasswordRetriever`
- Local state: `open`, `reason`, `test` (validation callback), `resolve`/`reject`, `password`, `showPassword`, `wasOriginallyFocused`

**Actions emitted:**
- Registers `passwordRetriever: (reason, test) => Promise<string>` via `setPasswordRetriever()`
- On submit: `test(password)` → if true, resolves with password → closes dialog → `onFocusRelinquished()` if not originally focused
- On abort: `reject()` → closes dialog → focus relinquish
- On X close: `reject(Error('User has closed password dialog'))`
- Toast: error on failed validation

**Conditional behavior:**
- Dialog visibility controlled by `open` state
- Password visibility toggle: `showPassword` → input type switches between 'password'/'text'
- Focus management: checks `isFocused()` (async) before requesting focus

**Edge cases:**
- Focus race condition mitigation: `isFocused()` is async promise
- Password NOT cleared on submit (stays in state)
- Multiple calls to `setPasswordRetriever` overwrite previous registration
- AutoFocus on TextField

**Frozen vs flexible:**
- **Frozen:** Promise-based API, focus management (check → request → relinquish), validation flow, focus toggle
- **Flexible:** Dialog title, input label, reason text, error messages, toggle icon

---

### RecoveryKeyHandler

**Component file:** `src/lib/components/RecoveryKeyHandler.tsx`

**Data consumed:**
- `WalletContext`: `managers`, `setRecoveryKeySaver`
- `Utils.toBase64()` from SDK
- Local state: `open`, `recoveryKey` (base64), `affirmative1`/`2`/`3`, `resolve`/`reject`, `copied`

**Actions emitted:**
- Registers `recoveryKeySaver: (key: number[]) => Promise<true>` via `setRecoveryKeySaver()`
- `navigator.clipboard.writeText(recoveryKey)` — copy, 2s feedback
- File download: blob creation → dynamic `<a>` link → `URL.revokeObjectURL()` cleanup
- `resolve(true)` — key saved
- `reject(Error('User abandoned...'))` — abandoned

**Conditional behavior:**
- Three-stage gated UI:
  - Stage 1 (`!affirmative1`): key display + copy/download + first checkbox
  - Stage 2 (`affirmative1`): warnings + checkboxes 2 and 3
  - Stage 3 (all three checked): "Save" button enabled
- Save button: `disabled={!affirmative1 || !affirmative2 || !affirmative3}`
- Copy button: `disabled={copied}` during 2s feedback

**Edge cases:**
- Copy timeout: potential memory leak if unmount during 2s timer
- File download cleanup: properly revokes blob URL
- Timestamp in file: `new Date()` (not ISO format)
- Key assumed valid (no validation on number array)

**Frozen vs flexible:**
- **Frozen:** Three-stage flow, three mandatory checkboxes (all required), copy/download patterns, promise API
- **Flexible:** Key display format, checkbox labels, warning text, button text, file naming

---

### FundingHandler

**Component file:** `src/lib/components/FundingHandler.tsx`

**Data consumed:**
- `WalletContext`: `setWalletFunder`, `network`
- `WalletInterface` instance
- Local state: `open`, `identityKey`, `paymentTX` (JSON string), `resolveFn`, `wallet`, `adminOriginator`, `tabValue`
- `WalletFundingFlow` sub-component (external)

**Actions emitted:**
- Registers funder via `setWalletFunder()`
- `wallet.getPublicKey({ identityKey: true })` — retrieve identity key
- `wallet.internalizeAction(payment, adminOriginator)` — manual (advanced) funding
- Tab navigation between Simple/Advanced
- `resolveFn()` on close (resolves promise)
- Toast: success/error on manual funding

**Conditional behavior:**
- Two tabs: Simple (0) → `<WalletFundingFlow>`, Advanced (1) → manual JSON input
- WalletFundingFlow: only renders if `wallet && adminOriginator`
- Funded button: `disabled={!paymentTX}`
- Identity key display: only in advanced tab if successfully retrieved

**Edge cases:**
- Identity key retrieval error: silently caught, empty string
- JSON parsing: `JSON.parse(paymentTX)` could throw → error toast
- Dual close paths: handleClose or WalletFundingFlow.onFundingComplete
- Tab selection resets on next dialog open

**Frozen vs flexible:**
- **Frozen:** Two-tab structure, simple/advanced split, identity key retrieval, wallet.internalizeAction call
- **Flexible:** Tab labels, JSON placeholder, button labels, success/error messages

---

## Ambiguous Items

### AMBIGUOUS: Unreachable Dead Code

**Files:**
- `src/lib/pages/Recovery/LostPassword.tsx` — exists but has no route in `UserInterface.tsx` or `Dashboard/index.tsx`. Contains incomplete TODO comments for phone submission and code verification. Functionality superseded by `RecoverPassword.tsx` which handles both WAB and self-custody paths.
- `src/lib/pages/Recovery/LostPhone.tsx` — exists but has no route. Contains incomplete phone-update stub. Phone number update not implemented in wallet manager.

**Recommendation:** Confirm whether these should be deleted or completed. If deleted, remove files. If completed, add routes.

### AMBIGUOUS: Undefined Theme Keys in `navigation/style.ts`

`navigation/style.ts` references the following keys not defined in `createTheme()`:
- `theme.palette.background.mainSection` → `undefined`
- `theme.palette.background.leftMenu` → `undefined`
- `theme.palette.background.leftMenuHover` → `undefined`
- `theme.palette.background.leftMenuSelected` → `undefined`
- `theme.palette.background.scrollbarTrack` → `undefined`
- `theme.palette.background.scrollbarThumb` → `undefined`
- `theme.maxContentWidth` → `undefined`

Impact: `content_wrap` and `page_container` classes still consumed by `Dashboard/index.tsx` via `makeStyles`. Background colors fall through to CSS defaults. Scrollbar styling is invisible. Menu component has been rewritten to use inline `sx` on Drawer, making `list_wrap` dead code.

**Recommendation:** During migration, replace these with proper Tailwind tokens. Do not attempt to replicate the undefined-value behavior.

### AMBIGUOUS: Hardcoded English Strings

Several handlers have hardcoded English strings outside of i18n:
- SpendingAuthorizationHandler: string matching on `"Increase spending limit"` and `"Create a spending limit"` for UI type detection
- Policy pages (`privacy.tsx`, `usage.tsx`): all content hardcoded English, not using `t()`

**Recommendation:** SpendingAuthorizationHandler string matching is behavioral (frozen) — the description comes from wallet-toolbox and must be matched as-is. Policy pages should be considered for i18n in a future pass but are not blocking for migration.

### AMBIGUOUS: Search Filter Disabled in MyIdentity

`MyIdentity/index.tsx` has a search filter that always returns `false` (lines 135-141), making the search box non-functional. This appears intentional or in-progress.

**Recommendation:** Preserve the disabled search during migration. Do not remove the search box UI (it may be completed later).

### AMBIGUOUS: `myIdentityKey` Hardcoded to 'self'

`Dashboard/index.tsx` line 71: `const [myIdentityKey] = useState('self')` with a TODO comment suggesting it should be fetched. This means the `/dashboard/counterparty/self` redirect goes to `/dashboard/counterparty/self` (no-op), which then resolves via IdentityClient in the CounterpartyAccess component.

**Recommendation:** Preserve current behavior during migration. The identity resolution happens downstream.
