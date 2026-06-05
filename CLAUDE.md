# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ProPaintball Pokladňa** is an offline-first paintball arena cashier/POS system for Slovak arenas. It is deployed as a **static file** — no build step, no server. The entire application lives in two HTML files:

- `index.html` (~7,675 lines) — the main cashier/POS/admin system
- `propaintball-kiosk.html` (~198 lines) — public arena kiosk for player registration

## Running the App

There is no build system, no `package.json`, no npm, and no compilation. To run locally:

```bash
python3 -m http.server 8080
# Then open http://localhost:8080/index.html
```

Opening `index.html` directly via `file://` works for most features, but Google APIs and Supabase require HTTP.

## Architecture

### Monolithic single-file SPA

All HTML, CSS, and JavaScript are embedded in `index.html`. No module system, no bundler, no framework — vanilla JS ES6+. CDN dependencies:
- `chart.js@4.4.0` — analytics charts
- `xlsx@0.18.5` — Excel export
- Google Fonts (Orbitron, Rajdhani)

### Data persistence (three layers)

1. **Browser `localStorage`** — primary store, offline-first. All reads/writes go through the `LS` helper (`index.html:1089`):
   - `LS.g(key, default)` — get and JSON-parse
   - `LS.s(key, value)` — JSON-stringify and set
   - `saveAll()` (`index.html:1097`) — persists all in-memory state at once

2. **Supabase REST API** — optional cloud backup/sync. Configured via `pb_sbu` (URL) and `pb_sbk` (API key). Key functions: `sbPush()` (`1622`), `sbPull()` (`1719`), `sbPollSessions()` (`1781`) — polls every **45 seconds** (not 30), also triggers on tab focus.

3. **Google Drive** — optional JSON backup. `gDriveConnect()` (`6057`) handles OAuth 2.0.

### In-memory state

All state lives in module-level variables loaded from localStorage on startup:

| Variable | Type | localStorage key |
|----------|------|-----------------|
| `prods` | `Product[]` | `pb_p` |
| `wbMap` | `{ [sessKey]: Workbench[] }` | `pb_wb` |
| `closed` | `Workbench[]` | `pb_cl` |
| `archive` | session array | `pb_arch` |
| `sessArr` | `Session[]` | `pb_ss` |
| `curSid` | string | `pb_cid` |
| `arenas` | `string[]` | `pb_ar` |
| `instructors` | `string[]` | `pb_ins` |
| `activities` | `string[]` | `pb_ac` |
| `biz` | object | `pb_biz` |
| `hrRate` | number | `pb_hr` |
| `admPw` | string | `pb_adm` |
| `sbUrl`, `sbKey` | string | `pb_sbu`, `pb_sbk` |
| `currentUser` | `{ name, role }` \| null | `pb_user` |

UI-only state (not persisted): `selId` (selected workbench ID), `payM`, `gPayM`, `pendGrp`, `chkd` (Set of checked bill IDs), `admUnlocked`.

### Key localStorage keys

| Key | Contents |
|-----|----------|
| `pb_p` | Products array |
| `pb_wb` | Workbench/bill map, keyed by session key |
| `pb_cl` | Closed bills |
| `pb_arch` | Archived sessions |
| `pb_ss` | Active sessions array |
| `pb_cid` | Current session ID |
| `pb_ins` | Instructors list |
| `pb_ar` | Arenas list |
| `pb_ac` | Activities list |
| `pb_customers` | Customer database |
| `pb_sbu` / `pb_sbk` | Supabase URL / API key |
| `pb_adm` | Admin password (default `admin123`) |
| `pb_hr` | Hourly rate for instructor payouts (default `6`) |
| `pb_biz` | Business info (name, ICO, DIC, address) |
| `pb_user` | Currently logged-in user `{ name, pw, role }` |
| `pb_audit` | Audit log entries |
| `pb_notes` | Session notes `{ [sessKey]: string }` |
| `pb_pouts` | Payouts `{ [sessKey]: { amt, costs, note, type } }` |
| `pb_deposits` | Deposits `{ [sessKey]: { amount, scope, wbId? } }` |
| `pb_custom_pws` | Custom instructor passwords `{ [name]: 'h_...' }` |
| `pb_lang` | Active language (`'sk'` or `'en'`) |
| `pb_gdrive_client_id` | Google Drive OAuth client ID |
| `pb_gdrive_token` / `pb_gdrive_token_exp` | Drive access token + expiry |
| `pb_cal_sa_creds` | Google Calendar service account credentials (JSON) |
| `pb_cal_id` | Google Calendar ID |
| `pb_migv` | Migration version (currently `2`) |

### Core data structures

```javascript
Session   { key: 'sess_' + Date.now(), date, time, arena, activity,
            instructors: string[], startedAt, note?,
            calEventId?, calRunNumber?, deposit? }

Workbench { id, name, email?, phone?, items[], sKey, openedAt, closedAt?,
            payM?: 'cash'|'card'|'invoice', paidExtras?: [],
            customerId?, signupId? }

// Item inside Workbench.items:
Item      { id, name, price, qty, disc: 0, isVoucher?, voucherNum? }

Product   { id, name, price, cat: 'Package'|'Ammo'|'Rental'|'Extra'|'Deposit',
            pkgId?, isAgency?, isDeposit? }

User      { name, pw, role: 'admin'|'instructor' }

Payout    { amt, costs, note, type: 'manual' }
```

**Session key** is always `'sess_' + Date.now()` — a string, unique per millisecond, used as the primary map key in `wbMap`.

### UI structure

Tabbed SPA: `#landing` entry screen → `#start` session-creation flow → `.shell` main layout.

Main tabs (render into `.page` divs):

| Tab | Render function | Line |
|-----|----------------|------|
| Pokladňa (cashier) | `rWbs()` + `rBill()` + `rPbt()` + `rDrk()` | 3099 / 3311 |
| Otvorené (open bills) | `rClosed()` filtered to open | 4230 |
| Správca (admin) | `rSpravca()` | 4762 |
| Analýzy (analytics) | `rAnalyzy()` | 5048 |
| Nastavenia (settings) | inline in HTML | — |

`rPbt()` (`2703`) renders ball/ammo tracking (hidden for Lasertag/Nerf).  
`rDrk()` (`2826`) renders the free-drink slot selector.  
Tab switching: `gT(tabName)` (`2285`).  
Selected workbench: `selWb(id)` (`3239`) sets `selId` and re-renders all cashier panels.

### Adding items to a bill

There is no `addItem()` function. Items are pushed directly onto `w.items` as `{ id, name, price, qty, disc: 0 }` inside `addProd(prodId)` (`2919`) and several other handlers. Always call `saveAll()` after.

### Agency billing

Package ID 5 (`Agentúra PB`) has `isAgency: true`. Agency wristbands differ from regular ones:
- Package itself billed as invoice; extras (ammo refills, rentals) billed separately as cash/card.
- Extras tracked in `w.paidExtras[]` array.
- Close button changes to "🧾 Balík na faktúru" and routes through `showAgencyPayModal()` (`3614`) → `confirmAgencyPayM()` instead of the standard `closeBill()` → `confirmPay()` (`3664`) path.
- `isAgencyWb(w)` (`1197`) detects agency wristbands via the package definition.

### Session close and payout

`openCloseSess(key, bills)` (`3689`) → user enters payout amount, optional costs (note required if costs > 0) → `confirmCloseSess()` (`3789`):
- Saves `{ amt, costs, note, type: 'manual' }` to `pb_pouts[sessKey]`.
- Saves deposit data to `pb_deposits[sessKey]` if present.
- Removes session from `sessArr` and `wbMap`, triggers cloud backup.

### Authentication

Roles: `admin` and `instructor`. Login: `loginUser(name, pw)` (`1123`). Passwords stored in localStorage — plain text in `pb_ins` defaults, or hashed via `simpleHash()` (`1238`) with prefix `h_` in `pb_custom_pws`. No server-side session; auth state lives in `pb_user`.

### NFC flow

WebSocket to `ws://{pb_nfc_ws_ip}:8765` populates `#mid` input. `_nfcProcess(raw)` (`2653`) strips separators, validates hex, formats UID as `AB:CD:EF`, then calls `assignWb(uid, name)` to create the wristband. Manual fallback: type UID in `#mid` and click assign.

### Internationalization

`LANG` object (`7221`) with `sk`/`en` keys. `t(key)` resolves to current language. Elements use `data-i18n` attributes. `toggleLang()` switches language and saves to `pb_lang`.

### External integrations

- **Supabase tables**: `pb_data` (backups), `pb_customers`, `pb_event_signups`
- **Google Calendar**: `sbPollSessions()` merges remote sessions into local state every 45 s; signup events auto-populate players
- **NFC reader**: optional WebSocket at `ws://localhost:8765`
- **Synthetic event IDs**: `{ARENA_CODE}_{ACTIVITY_CODE}_{DATE}` (e.g. `RC_PB_20260427`) tie Calendar events to sessions without manual config

### Modal system

`om(id)` (`5755`) / `cm(id)` (`5773`) open/close modals by toggling the `.op` class on `.mw` wrappers. Backdrop click closes. Known modals: `m-pay`, `m-epay`, `m-grp`, `m-adm`, `m-item`, `m-wb`, `m-sess`, `m-vrp`, `m-reopen`, `m-drink`, `m-xh`, `m-pout`, `m-note`, `m-chart`, `m-edates`, `m-report`, `m-del-sess`, `m-backup`, `m-restore-confirm`, `m-agency-pay`, `m-deposit`, `m-close-sess`.

## Conventions

### Navigating the file

Everything is in one file — use line-number anchors or `grep`. Function names use camelCase; CSS uses single-letter modifier classes: `.R` red/danger, `.G` green, `.B` blue, `.A` amber, `.T` teal, `.P` purple.

### CSS variables

All colors and spacing are CSS custom properties in `:root` around line 56. Never hardcode colors — always use variables like `var(--red)`, `var(--grn)`, `var(--bg2)`.

### Saving state

Always call `saveAll()` after mutating any in-memory state (`prods`, `wbMap`, `closed`, `sessArr`, etc.).

### `escHtml(str)` (`1237`)

All user-supplied strings rendered into `.innerHTML` must be wrapped with `escHtml()`.

### Key utility functions

| Function | Line | Purpose |
|----------|------|---------|
| `fp(v)` | 1236 | Format price → `€1.50` |
| `fD(iso)` | ~1240 | Format date → `01. 06. 2026` |
| `fT(iso)` | ~1248 | Format time → `14:30` |
| `fDT(iso)` | ~1255 | Format datetime |
| `toast(msg)` | 5807 | Show floating notification for 2.8 s |
| `dot(icon, color)` | 1486 | Set `#sdot` status indicator |
| `escHtml(s)` | 1237 | HTML entity escape |
| `om(id)` / `cm(id)` | 5755/5773 | Open/close modal |
| `$(id)` | ~1180 | `document.getElementById` shorthand |
| `gP(name)` | ~1182 | `querySelector` by name attribute |

### Adding a product category or payment method

Product categories: `'Package'|'Ammo'|'Rental'|'Extra'|'Deposit'`. Payment methods: `'cash'|'card'|'invoice'`. Adding a new value requires auditing all `switch`/`if` blocks that enumerate these strings.

### Supabase API key handling

`SB_PUB_URL` / `SB_PUB_KEY` (`6165`) are an intentionally public Supabase anon key scoped to `pb_event_signups` for the kiosk. The private admin key is user-configurable (`pb_sbk`) and never hardcoded. Do not embed new secrets in source.
