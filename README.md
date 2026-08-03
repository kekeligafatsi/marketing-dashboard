# Marketing Dashboard

A single-page sales and lead pipeline tracker built for the Dekells/GRiT sales and marketing team. It gives the team a shared, no-login dashboard for logging prospects and clients, tracking deal status and follow-up dates, watching progress against a monthly revenue and lead target, and seeing a running activity feed of who changed what. It is designed to be opened directly as a file or hosted as a static page — anyone with the link can view and edit the shared pipeline instantly.

## Key features

- **Prospect/client pipeline** — add, edit, and soft-delete entries with company, contact person, phone, email, deal value, status (Contacted / Quoted / Follow-up due / Won / Lost), notes, last-contact date, next-follow-up date, and who logged the entry.
- **Search, filter, and sort** — free-text search across company/contact/phone/email, filter by status, and sort by next follow-up date, company, status, or deal value.
- **Due-today/this-week list** — automatically surfaces entries that are overdue or due within the next 7 days, flagged visually (red for overdue, amber for due soon).
- **Monthly targets with progress bars** — editable shared revenue target (GH₵) and lead-count target, each with a live progress bar computed from the current month's actual won revenue / logged leads.
- **Stat tiles** — deals won this month, win rate, overdue count, lead target, revenue target, and open-deal count.
- **Trash with undo** — deleting an entry moves it to a trash list (last 20) instead of destroying it, with a one-click restore.
- **Activity feed** — a running, timestamped log (last 50 entries) of adds/edits/deletes/restores, attributed to whoever is logged in the entry's "Logged By" field.
- **Month-over-month history** — on first load each month, the previous month's target vs. actual (revenue and leads) is automatically archived (last 6 months kept).
- **One-click outreach links** — each pipeline row generates a pre-filled WhatsApp (`wa.me`) link and a `mailto:` link addressed to that contact.
- **Optional live shared sync** — if a Supabase project is configured, all state syncs to a shared `dashboard_state` table so multiple people see the same live data; otherwise it falls back to per-browser `localStorage`.

## Tech stack

This is a single self-contained HTML file — vanilla JavaScript, inline CSS, no framework, and no build step. The only external dependency is the Supabase JS client, loaded from a CDN `<script>` tag, used purely as an optional real-time-ish persistence backend. See [TECHSTACK.md](TECHSTACK.md) for full details.

## Setup / run instructions

There is no build step and no package manager involved.

- **Just open it**: double-click `index.html` or open it in a browser (`File > Open`). It works standalone with `localStorage` only.
- **To enable shared live sync**: point the `window.MARKETING_DASHBOARD_CONFIG` object near the top of the `<script>` block in `index.html` at your own Supabase project's URL and anon key, and create a `dashboard_state` table with `key` (text, primary/unique), `value` (jsonb), and `updated_at` (timestamptz) columns. The file currently ships with a project already wired in for the team's shared use.
- **To host it**: upload `index.html` to any static host (GitHub Pages, Netlify, S3, etc.) — no server-side code is required.

There are no npm scripts, no tests, and no CI configuration in this project.

## Project structure

The project is a single file: `index.html`. There is no folder hierarchy of components, hooks, or services — everything (markup, styles, state, rendering, and Supabase sync) lives in that one file. See [TREEVIEW.md](TREEVIEW.md) for an annotated breakdown of its internal sections, [TECHSTACK.md](TECHSTACK.md) for the technical detail, and [ARCHITECTURE.md](ARCHITECTURE.md) for how data flows between the page, `localStorage`, and Supabase (including the multi-editor conflict handling).

This project does not have an `API.md` (no backend routes are exposed — Supabase is used purely as a key-value store), a `COMPONENTS.md` (no component framework; see the section table in [TREEVIEW.md](TREEVIEW.md#body--ui-sections-the-blocks) instead), or an `.env.example` (no environment variables are read — the one piece of config is a hardcoded object in `index.html`, see below).

## Data and persistence

- **Primary store**: browser `localStorage` under the key `marketing-dashboard-state-v1`, holding the entire app state (targets, entries, activity feed, trash, and history) as JSON.
- **Optional shared store**: if `window.MARKETING_DASHBOARD_CONFIG` has a Supabase URL and anon key, the same state JSON is also read from and written to a single row (`key = 'marketing-dashboard'`) in the `dashboard_state` table on load and after every change, so multiple people editing the page see a shared, converging state.
- All data loaded from either source is passed through a sanitizer (`sanitizeState`/`sanitizeEntry`) before use, so malformed or unexpected JSON (from a corrupted local copy, or a write via the public anon key) can't crash the page.

## Contributing

There's no separate build/lint/test tooling to run — see [CONTRIBUTING.md](CONTRIBUTING.md) for local setup, the manual test checklist used in place of an automated suite, and the PR process.

## License

No license file is currently present in this repository.
