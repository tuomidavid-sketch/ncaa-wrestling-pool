# CLAUDE.md — NCAA Wrestling Pool

This file provides guidance for AI assistants (Claude and others) working on this codebase.

## Project Overview

A single-page web application for running a family NCAA Wrestling Tournament pool. Users draft wrestlers within a budget, earn points based on tournament performance, and compete on a leaderboard.

- **2026 NCAA Wrestling Championships** — Cleveland, Ohio — March 19–21, 2026
- **Backend**: Supabase (PostgreSQL + REST API)
- **Frontend**: Vanilla JS, HTML, CSS — no build tooling, no frameworks, no npm

## Repository Structure

```
ncaa-wrestling-pool/
├── index.html      # Entire application (HTML + CSS + JS, ~742 lines)
└── README.md       # Brief project description
```

There is no `package.json`, no `node_modules`, no build step, and no test suite. The app runs directly in the browser as a static HTML file.

## Architecture

### Single-File SPA

Everything lives in `index.html`:
- **Lines 1–148**: HTML structure and embedded `<style>` CSS
- **Lines 149–742**: Embedded `<script>` JavaScript (~594 lines)

### Backend: Supabase REST API

Three tables accessed via `fetch` with Supabase headers:

| Table | Purpose |
|-------|---------|
| `players` | User pool entries (`id`, `name`, picks as JSON) |
| `scores` | Wrestler point totals (`name`, `points`) |
| `meta` | Key-value store (`lastRefresh`, `brackets` JSON) |

Core helpers (defined in `index.html`):
- `sbSelect(table)` — `GET /rest/v1/{table}?select=*`
- `sbUpsert(table, rows)` — `POST` with `Prefer: resolution=merge-duplicates`
- `sbDeleteWhere(table, col, op, val)` — `DELETE /rest/v1/{table}?{col}={op}.{val}`

The Supabase project URL and anon key are hardcoded in `index.html`. Do not move these to environment variables unless also adding a build step.

### Global State

```js
let myPicks={}, myName='', myId=null   // current user
let allPlayers=[], allScores={}, lr=null
let allBrackets={}  // wrestler -> 'champ' | 'conso' | 'out'
```

State is persisted to `localStorage` under the key `ncaa_pool_v3` via `ldLocal()` / `svLocal()`.

## Key Domain Concepts

### Weight Classes

```js
const WTS = ['125','133','141','149','157','165','174','184','197','285']
```

One wrestler must be picked per weight class. Each wrestler has a seed-based cost (1–32 pts). Total budget is **200 points**.

### Scoring

| Placement | Points |
|-----------|--------|
| 1st | 16 |
| 2nd | 12 |
| 3rd | 10 |
| 4th | 9 |
| 5th | 7 |
| 6th | 6 |
| 7th | 4 |
| 8th | 3 |

Bonus points per win (added to a wrestler's running score):
- Championship bracket win: **+1 pt**
- Consolation bracket win: **+0.5 pt**
- Fall (pin): **+2 pt**
- Technical fall: **+1.5 pt**
- Major decision: **+1 pt**

Championship finalists receive placement points only (no advancement bonus for the final match).

### Bracket Status (`allBrackets`)

Each wrestler is tagged: `'champ'` (still in championship), `'conso'` (consolation bracket), or `'out'` (eliminated). This drives color coding in the UI.

### Baked Data

Tournament data is embedded directly in `index.html` as `const` objects:
- `BAKED_SCORES` — final/current point totals per wrestler
- `BAKED_BRACKETS` — bracket status per wrestler
- `BAKED_REFRESH` — ISO timestamp of last data refresh
- `W` — wrestler rosters indexed by weight class

When updating scores mid-tournament, update these constants and push to Supabase via the Commissioner Panel.

## UI Structure

Tab-based navigation. Active tab is stored in a `data-t` attribute and toggled via click listeners.

| Tab ID | View |
|--------|------|
| `p-lb` | Leaderboard (player rankings) |
| `p-dr` | Draft / My Picks |
| `p-sc` | All wrestler scores |
| `p-gr` | Grid view (table) |

### Commissioner Panel

Hidden UI accessible by triple-tapping the settings icon. Allows:
- Pasting championship/consolation results text to parse and push scores
- Clearing all scores

This is the intended workflow for updating live tournament results.

## CSS Conventions

CSS variables defined at `:root`:
```css
--bg:   #08090f   /* dark background */
--gold: #c9a227   /* primary accent */
--grn:  #2ecc71   /* champion / success */
--red:  #e74c3c   /* eliminated / error */
--blu:  #3b82f6   /* secondary / info */
```

Mobile-first responsive design using media queries. No CSS framework.

## Development Workflow

### Making Changes

1. Edit `index.html` directly — this is the entire app.
2. Open in a browser to test (no server required for most features).
3. Supabase calls will require network access; test with a live browser session.

### Deploying

Commit and push `index.html`. No build step needed. The file is served as-is.

### Updating Tournament Scores

Use the Commissioner Panel in the live app, or:
1. Update `BAKED_SCORES`, `BAKED_BRACKETS`, and `BAKED_REFRESH` constants in `index.html`
2. Push updates to Supabase via `sbUpsert`
3. Commit the updated `index.html`

### Git Branch Convention

Feature branches follow the pattern `claude/<description>-<id>`. Always push to the designated branch and open a PR against `main` (not `master`).

## What to Avoid

- **Do not introduce a build toolchain** unless explicitly requested — the zero-dependency approach is intentional.
- **Do not split the file** into separate JS/CSS files unless the project structure is being intentionally restructured.
- **Do not add npm packages** without also adding `package.json` and a build step.
- **Do not rotate the Supabase anon key** without updating it in `index.html`.
- **Do not add a test framework** without discussion — there is currently no testing infrastructure.

## No Automated Tests

There are no unit, integration, or e2e tests. Verify changes manually in the browser. When making logic changes (especially to scoring or budget calculations), trace through the affected code paths carefully.
