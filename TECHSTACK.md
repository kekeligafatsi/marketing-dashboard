# Tech Stack

## Runtime / language

Plain JavaScript (ES2020+ syntax: optional chaining, `structuredClone`, `crypto.randomUUID`) running directly in the browser. No TypeScript, no transpilation, no bundler. HTML5 and CSS3, both authored inline inside `index.html`.

## Framework / build tooling

None. This is a single self-contained HTML file with a `<style>` block and a `<script>` block — there is no `package.json`, no lockfile, no bundler config (Vite/Webpack/etc.), no `tsconfig.json`, and no dev server required. The file is meant to be opened directly or served as a static asset. This is a deliberate architectural choice for a small internal tool: zero install friction for non-technical users sharing a single link.

## Key libraries

- **`@supabase/supabase-js@2`** — loaded from the jsDelivr CDN (`https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2`). Used exclusively as an optional persistence/sync layer: creating a client with `window.supabase.createClient(...)`, and reading/writing a single row in the `dashboard_state` table. No other Supabase features (auth, storage, realtime channels) are used — `auth: { persistSession: false }` is explicitly set since the app has no login.

No other third-party JS libraries, CSS frameworks, icon sets, or fonts are used — fonts fall back to the system stack (`Arial, Helvetica, sans-serif`).

## Styling approach

Hand-written CSS inside a single `<style>` block, using:
- CSS custom properties (`:root { --navy, --blue, --danger, --card, ... }`) for a consistent dark navy/blue color palette (background gradient, card surfaces, status colors).
- Flexbox and CSS Grid for layout (`.grid`, `.layout`, `.target-row`, `.toolbar`, `.form-grid`), with a single responsive breakpoint at `max-width: 980px` that collapses multi-column grids to one column and hides two table columns on small screens.
- Utility-ish reusable classes rather than a framework (`.card`, `.pill`, `.status-badge`, `.subtle`, `.small`) — no Tailwind, no CSS Modules, no CSS-in-JS.
- A handful of inline `style="..."` attributes directly in the HTML for one-off elements (e.g. the sync status banner, target inputs).

## State management pattern

A single mutable module-level object, `state`, holds the entire application state (`revenueTarget`, `leadTarget`, `entries[]`, `activity[]`, `trash[]`, `history[]`). There is no reactive framework or virtual DOM:
- All mutations are plain JS (`state.entries.unshift(...)`, `state.entries = state.entries.filter(...)`, etc.) performed directly inside event handler functions.
- After any mutation, `saveState()` is called, which persists to `localStorage`, optionally pushes to Supabase, and calls `render()`.
- `render()` is a full manual re-render: it re-reads `state` and rewrites the innerHTML of each section (`renderStats`, `renderProgress`, `renderDueList`, `renderTrash`, `renderActivity`, `renderHistory`, `renderTable`) — there is no diffing; each render call fully regenerates that section's markup and re-binds its event listeners (e.g. edit/delete buttons in the pipeline table, undo buttons in trash).
- A small set of transient UI-only variables (`filterText`, `filterStatus`, `sortField`) live outside `state` and are not persisted — they only affect what `renderTable()` displays.
- `structuredClone` is used to produce independent copies of `DEFAULT_STATE` so the constant template is never mutated in place.

## Backend / services

There is no custom backend/API server. The only backend is Supabase (a hosted Postgres + REST/Realtime service), used purely as a key-value document store:

- **Table**: `dashboard_state`, with (inferred from the queries) at least `key` (text), `value` (jsonb — the entire serialized `state` object), and `updated_at` (timestamptz).
- **Read**: on load, `select('value, updated_at').eq('key', 'marketing-dashboard').maybeSingle()` fetches the shared row; if present, it replaces local state (after sanitization).
- **Write**: `upsert(payload, { onConflict: 'key' })` writes `{ key: 'marketing-dashboard', value: state, updated_at }` after every local state change (via `persistToSupabase()`), so the shared row always reflects the last writer's full state.
- **Concurrency handling**: before writing, the app re-checks the row's current `updated_at` against the last value it saw (`lastKnownRemoteUpdatedAt`). If someone else has written in the meantime, the local page discards its pending write, pulls their newer state instead, re-renders, and shows a toast telling the user to redo their edit — a simple optimistic-concurrency guard against silently clobbering a concurrent editor.
- **Failure handling**: sync failures (network issues, RLS/permission errors, etc.) are caught, logged to `console.warn`, and surfaced via a persistent on-page banner (`#syncStatus`) reading "Sync issue — your changes are saved locally and will retry automatically on the next edit." The app never gives up on syncing permanently — every `saveState()` call retries.
- **Configuration**: the Supabase project URL and anon key are set inline in `window.MARKETING_DASHBOARD_CONFIG` at the top of the script. Because this is a public anon key embedded in client-side HTML, the app treats all remote data as untrusted input and sanitizes it (see below) before ever rendering or storing it.

## Data validation / sanitization

- `sanitizeEntry(raw)` and `sanitizeState(raw)` coerce and validate every field of incoming JSON (from `localStorage` or Supabase) to expected types, rejecting entries missing a `company`/`contactPerson`, clamping deal values to non-negative finite numbers, restricting `status` to the known `VALID_STATUSES` list, capping `activity` to 50 items and `trash` to 20 items, and generating a fresh UUID for any missing/invalid `id`. This guards against a corrupted local copy or a malicious/malformed write via the public anon key from ever crashing rendering or injecting unexpected shapes.
- `escapeHtml(value)` is applied to all user-supplied strings before they're interpolated into template-literal HTML (table cells, due-list items, trash items, activity feed), preventing stored/reflected XSS since deal/contact data can be entered by anyone with the link.

## Linting / testing

None present — no ESLint/Prettier config, no test runner, no CI workflow files.

## Notable architectural decisions

- **Single self-contained HTML file, no build step** — chosen for a small internal tool that needs to be trivially shareable and editable without tooling.
- **`localStorage`-first with best-effort Supabase sync** — the app always works offline/standalone; Supabase is strictly additive and its absence or failure degrades gracefully to local-only persistence.
- **Optimistic concurrency via `updated_at` comparison** rather than realtime subscriptions or CRDT merging — simple, good enough for a small team's low-frequency edits, at the cost of a coarse "last full write wins, unless someone else wrote more recently" model (the whole state blob is overwritten each save, not merged field-by-field).
- **Full manual re-render per state change** rather than a virtual DOM/reactive framework — appropriate given the app's modest UI complexity and the goal of zero dependencies.
- **Soft delete + capped trash/activity/history arrays** — deliberate bounded-memory design (trash capped at 20, activity at 50, history at 6 months) so the state document doesn't grow unbounded in `localStorage`/Postgres over time.
