# Bowen's Business - AI Coding Agent Instructions

## Project overview
A single-file HTML baby tracker app for two users. Tracks feeds (ml, EBM vs formula ratio), nappies (wet/dirty/both), and pumping (ml). Shared data between two devices via Cloudflare Worker → Supabase.

## Architecture
```
index.html (single file app)
    ↓ fetch + X-App-Secret header
Cloudflare Worker (crimson-field-cfbb.backupjwb.workers.dev)
    ↓ Supabase REST API
Supabase (iixfidxyrkvjxpnlxpit.supabase.co)
    └── entries table: id (int), data (jsonb)
        ├── id=1 → live data: { entries: [...] }
        └── id=2 → daily snapshot backup
```

## Key constraints
- **Single file**: all HTML, CSS, and JS lives in `index.html`. No build step, no bundler.
- **No ES6+ in app JS**: use `var`, standard `function` declarations, and string concatenation throughout. Arrow functions, template literals, `const`/`let`, and special characters (·, ←, →) in JS strings have caused syntax errors in the artifact preview environment. Use Unicode escapes (`\u00b7`, `\u2190`, `\u2192`) instead.
- **Worker uses ES modules**: the Cloudflare Worker (`worker.js`) uses `export default` and modern JS - this is fine, it runs in a different environment.
- **No localStorage writes on server failure**: if `serverConnected === false`, saves are blocked entirely. Never fall back to writing stale localStorage data to the server.

## Data model
All entries stored as a flat array, sorted newest-first by `ts` (Unix ms timestamp):
```json
{ "type": "feed", "ml": 90, "ebm": 45, "formula": 45, "ts": 1775033400000, "who": "Tim" }
{ "type": "nappy", "nappyType": "wet|dirty|both", "ts": 1775033400000, "who": "Bob" }
{ "type": "pump", "ml": 100, "ts": 1775033400000, "who": "Keith" }
```

## Critical state variables
- `dataLoaded` — set true only after a successful server fetch or localStorage fallback. Blocks saves until true.
- `serverConnected` — set true only after a successful server fetch. Blocks saves if false. Set false on any save or fetch failure.
- `entriesVersion` — incremented on every entries mutation. Used to avoid unnecessary chart redraws.
- `cachedPrevMap` — cached Map of entry → previous same-type entry timestamp. Invalidated on any mutation.

## Timezone handling
- `nowLocal()` subtracts `getTimezoneOffset() * 60000` before calling `.toISOString()` to correctly populate datetime-local inputs in BST.
- `tsFromInput(id)` uses `new Date(v).getTime()` directly — the browser correctly interprets datetime-local values as local time. Do NOT add timezone offset on read.
- Never use `.toISOString()` directly for display or input population — it always outputs UTC.

## Deployment
- App hosted on GitHub Pages (public repo)
- Worker deployed via Cloudflare dashboard (paste and deploy, no CLI)
- No CI/CD — manual copy/paste deploy from GitHub browser editor or VS Code push
- Secrets (`APP_SECRET`, `SUPABASE_KEY`, `SUPABASE_URL`) stored as Cloudflare Worker environment variables, never in the HTML

## Patterns to follow
- All DOM string building uses concatenation: `"<div>" + variable + "</div>"` — no template literals
- Chart instances stored in `charts` object, destroyed before recreation via `mkChart(id, config)`
- Filters (`logFilter`, `logTypeFilter`) are independent — AND logic, not OR
- Pagination at 20 entries per page (`PAGE_SIZE = 20`)
- All entry mutations must: update `entries`, sort descending by `ts`, set `cachedPrevMap = null`, increment `entriesVersion`, then call `saveData()`

## Known gotchas
- Supabase free tier pauses after 1 week of inactivity — first request after pause will fail, triggering offline mode
- JSONBin (previous backend) had a 10k requests/month limit and aggressive caching — do not revert to it
- The app previously suffered two data loss incidents from race conditions and stale cache overwrites — the `serverConnected` guard is the primary protection against recurrence
- Cron trigger in Cloudflare runs at 02:00 daily, snapshots live data (id=1) to backup row (id=2)
