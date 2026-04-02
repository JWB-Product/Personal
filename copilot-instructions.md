# Bowen's Business - AI Coding Agent Instructions

## Project overview
Single-file HTML baby tracker app. Tracks feeds, nappies, and pumps. Data sync via Cloudflare Worker to Supabase. Local cache fallback in localStorage.

## Architecture
```
index.html (single file app)
  - local UI (feed, nappy, pump)
  - local cache: ll_entries (JSON array)
  - POST/PUT/DELETE to worker endpoint via fetch + X-App-Secret
Cloudflare Worker (crimson-field-cfbb.backupjwb.workers.dev)
  - authenticates X-App-Secret against env.APP_SECRET
  - proxies to Supabase REST API (/rest/v1/entries)
  - handles GET/POST/PUT/DELETE with row-per-entry semantics
Supabase (iixfidxyrkvjxpnlxpit.supabase.co)
  - table: public.entries
    columns: id (uuid pk), type (text), ml (int4), ebm (int4), formula (int4), nappy_type (text), ts (int8), who (text), created_at (timestamptz), updated_at (timestamptz)
```

## Key constraints
- Single-file: all UI logic in `baby-tracker-html.html`, no bundler.
- Keep app JS ES5-compatible: `var`, `function`, string concatenation. No `let/const`, templates, arrow functions.
- Worker can use modern JS (ES module), separate deployment.
- When `serverConnected === false`, do not send mutations to server; block with error and stop local save.

## Data model
App object shape:
- feed: `{type:'feed', id:'uuid', ml:100, ebm:50, formula:50, ts:<ms>, who:'name'}`
- pump: `{type:'pump', id:'uuid', ml:100, ts:<ms>, who:'name'}`
- nappy: `{type:'nappy', id:'uuid', nappyType:'wet|dirty|both', ts:<ms>, who:'name'}`

Supabase row shape (worker converts before send):
- `{id, type, ml, ebm, formula, nappy_type, ts, who}`

## Important state
- `dataLoaded` true only after successful initial load from server or localStorage fallback.
- `serverConnected` true only when server is reachable; set false on API errors.
- `entries` array of normalized entries, sorted descending by `ts`.
- `entriesVersion` for redraw control (charts/trends).
- `cachedPrevMap` map for nappy/feed gap calcs.

## Time handling
- `nowLocal()`: local datetime-local string generation using UTC offset via `getTimezoneOffset()` subtract.
- `tsFromInput(id)`: `new Date(inputValue).getTime()` (local date-time parse), no extra offset.
- On edit modal, input initialized via `new Date(entry.ts - offset).toISOString().slice(0,16)` so local value matches stored UTC instant.

## Save flow
1. Create entry object in UI save functions.
2. `saveEntry()` sets `entry.id` (UUID) if missing.
3. `upsertEntryOnServer()` maps to Supabase shape:
   - `nappyType` -> `nappy_type`
   - `ml`, `ebm`, `formula`, etc.
4. `apiFetch()` POST/PUT to worker with `X-App-Secret`, JSON body.
5. On success, update `entries`, sort by `ts` desc, `persistEntriesToLocal()`, `renderHome()`, `renderRecent()`, optionally `renderTrends()`.
6. On failure, set `serverConnected=false`, show error toast, stop writes.

## Log and filters
- `getFiltered()` applies date and type filters from UI.
- `renderRecent()` uses `entries` derived from `getFiltered()` with pagination.
- All log data is newest-first: `entries.sort((a,b)=>b.ts-a.ts)` on every mutation/load.
- `buildPrevMap()` uses reversed entries stable traversal.

## Common gotchas
- `nappyType` in JS vs `nappy_type` in DB mapping is critical.
- API 400 can happen if column names mismatch (the worker currently expects strongly typed Supabase columns).
- `ts` is int64 Unix ms, not ISO string.
- local timezone conversion off-by-one can produce future label and false `just now`.

## Deployment notes
- Cloudflare Worker must be updated if Supabase table schema changes.
- Worker should return path-specific errors and `details` from Supabase for debugging.
- Cron backup to row id=2 for snapshot is still referenced in model but may not be in this version.
