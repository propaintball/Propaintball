# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ProPaintball Pokladňa** is an offline-first paintball arena cashier/POS system for Slovak arenas. It is deployed as a **static file** — no build step, no server. The entire application lives in two HTML files:

- `index.html` (~7,675 lines) — the main cashier/POS/admin system
- `propaintball-kiosk.html` (~198 lines) — public arena kiosk for player registration

## Running the App

There is no build system, no `package.json`, no npm, and no compilation. To run locally:

```bash
# Any static HTTP server works, e.g.:
python3 -m http.server 8080
# Then open http://localhost:8080/index.html
```

Opening `index.html` directly in a browser (via `file://`) also works for most features, but some PWA and CORS-related features (Google APIs, Supabase) require HTTP.

## Architecture

### Monolithic single-file SPA

All HTML, CSS, and JavaScript are embedded in `index.html`. There is no module system, no bundler, and no framework — vanilla JS ES6+. External dependencies are loaded from CDN:
- `chart.js@4.4.0` — analytics charts
- `xlsx@0.18.5` — Excel export
- Google Fonts (Orbitron, Rajdhani)

### Data persistence (three layers)

1. **Browser `localStorage`** — primary store, offline-first. All reads/writes go through the `LS` helper (`index.html:1089`):
   - `LS.g(key, default)` — get and JSON-parse
   - `LS.s(key, value)` — JSON-stringify and set
   - `saveAll()` (`index.html:1097`) — persists all in-memory state to localStorage at once

2. **Supabase REST API** — optional cloud backup/sync. Configured via `pb_sbu` (URL) and `pb_sbk` (API key) stored in localStorage. Key functions: `sbPush()` (`index.html:1622`), `sbPull()` (`index.html:1719`), `sbPollSessions()` for real-time signup polling.

3. **Google Drive** — optional JSON backup. `gDriveConnect()` (`index.html:6057`) handles OAuth 2.0.

### Key localStorage keys

| Key | Contents |
|-----|----------|
| `pb_p` | Products array |
| `pb_wb` | Workbench/bill map (active players), keyed by session key |
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

### Core data structures

```javascript
Session   { key, date, time, arena, activity, instructor[], calEventId?, deposit? }
Workbench { id, name, email?, phone?, items[], sKey, openedAt, closedAt?,
            payM?: 'cash'|'card'|'invoice', customerId?, signupId? }
Product   { id, name, price, cat: 'Package'|'Ammo'|'Rental'|'Extra'|'Deposit',
            pkgId?, isAgency?, isDeposit? }
User      { name, pw, role: 'admin'|'instructor' }
```

### UI structure

The app is a tabbed SPA with a `#landing` entry screen, a `#start` session-creation flow, and a `.shell` main layout. The five main tabs render into `.page` divs:

- **Pokladňa** (cashier) — `rWbs()` (`index.html:3099`), `rBill()` (`index.html:3311`)
- **Otvorené** — open bills
- **Správca** — `rSpravca()` (`index.html:4762`), admin session management
- **Analýzy** — `rAnalyzy()` (`index.html:5048`), revenue/chart reports
- **Nastavenia** — settings, backup, cloud config

Tab switching: `gT(tabName)` (`index.html:2285`).

### Authentication

Roles: `admin` and `instructor`. Login: `loginUser(name, pw)` (`index.html:1123`). Passwords are stored in localStorage as plain text or with a custom weak hash (`simpleHash()`, `index.html:1238`) using the prefix `h_`. There is no server-side session — auth state lives in `pb_user` in localStorage.

### Internationalization

UI strings are in the `LANG` object (`index.html:7221`) with `sk`/`en` keys. HTML elements use `data-i18n` attributes; `t(key)` resolves to the current language string. `toggleLang()` switches the active language.

### External integrations

- **Supabase tables**: `pb_data` (backups), `pb_customers`, `pb_event_signups`
- **Google Calendar**: auto-populates players from event signups via 30-second polling
- **NFC reader**: optional WebSocket at `ws://localhost:8765` for card scanning
- **Synthetic event IDs**: `{ARENA_CODE}_{ACTIVITY_CODE}_{DATE}` (e.g. `RC_PB_20260427`) tie Calendar events to sessions without manual config

## Conventions

### Modifying the app

Since everything is in one file, use line-number anchors from this file or `grep` to navigate. Function names use camelCase; local CSS uses short single-letter modifier classes (`.R` = red/danger, `.G` = green, `.B` = blue, `.A` = amber, `.T` = teal, `.P` = purple).

### CSS variables

All colors and spacing are defined as CSS custom properties at the top of the `<style>` block (`:root`, around line 56). Always use these variables — never hardcode colors.

### Adding a product category or payment method

Product categories are the string union `'Package'|'Ammo'|'Rental'|'Extra'|'Deposit'`. Payment methods are `'cash'|'card'|'invoice'`. If you add a new value, audit all `switch`/`if` blocks that enumerate these strings.

### Saving state

Always call `saveAll()` after mutating any in-memory state (`prods`, `wbMap`, `closed`, `sessArr`, etc.) to persist changes to localStorage.

### `escHtml(str)`

All user-supplied strings rendered into `.innerHTML` must be wrapped with `escHtml()` to prevent XSS.

### Supabase API key handling

`SB_PUB_KEY` (`index.html:6165`) is an intentionally public Supabase anon key scoped to the `pb_event_signups` table for the kiosk flow. The private admin key is user-configurable (`pb_sbk`) and never hardcoded. Do not embed new secrets in source.
