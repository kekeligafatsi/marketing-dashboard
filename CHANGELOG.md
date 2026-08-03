# Changelog

All notable changes to this project are documented in this file, in the [Keep a Changelog](https://keepachangelog.com/) format.

This project has a single file (`index.html`) and a small git history: one commit, `44cc131` ("Initial marketing dashboard", 2026-07-23). The working copy currently contains additional uncommitted changes (a fix pass and a UI redesign) on top of that commit; those are listed under **[Unreleased]** below. Entries are derived from the actual commit and from a diff of the working tree against it — nothing here is fabricated.

## [Unreleased]

Uncommitted changes currently in the working copy, not yet part of a git commit.

### Added
- Left sidebar navigation (`.sidebar`, `.side-btn`) matching the `dekells-manager` app's nav pattern, splitting the dashboard into three pages (Dashboard / Pipeline / History) instead of one long scrolling column.
- Trash moved from an always-visible inline card to an icon-triggered slide-over panel (`#trashBtn` → `#trashPanel`), with a red badge dot shown only when the trash is non-empty. Closes via the Close button, backdrop click, or Escape.
- Activity Feed moved from an always-visible inline card to an icon-triggered dropdown panel (`#activityBtn` → `#activityDropdown`), with the same badge-dot treatment. Closes via outside click or Escape. Both the trash and activity buttons expose `aria-haspopup`/`aria-expanded` for screen readers.
- Dekells brand palette (`:root` color tokens — navy `#00094B`, orange `#ff5e1f`, plus the supporting grey/status colors) ported from `dekells-manager-v9.html` for visual consistency across Dekells' internal tools, replacing the previous dark navy/blue gradient theme.
- Sync status banner (`#syncStatus`) that appears when a Supabase write or read fails, telling the user their changes are saved locally and will retry automatically.
- Input sanitization layer (`sanitizeEntry`, `sanitizeState`, `VALID_STATUSES`) applied to any state loaded from `localStorage` or Supabase, so corrupted or unexpected data can no longer crash rendering or smuggle in invalid fields.
- HTML-escaping (`escapeHtml`) applied to all user-entered values before they're rendered (pipeline table, due list, trash list, activity feed), closing an XSS gap where a company name, contact name, or note containing HTML/script content would previously be rendered unescaped.
- Optimistic-concurrency check before every Supabase write: the app compares the row's live `updated_at` against the last value it fetched, and if another editor has written since, it discards the pending local write, pulls their newer state, re-renders, and shows a toast asking the user to redo their edit — instead of silently overwriting a concurrent change.
- `wonAt` timestamp is now set/preserved explicitly when an entry's status changes to "won" (and cleared if it's changed away from "won"), instead of being inferred from `updatedAt`.

### Changed
- Trash capacity raised from 5 to 20 retained entries before the oldest is permanently purged.
- Deal-value validation on the entry form now rejects negative or non-finite values (previously only rejected falsy/zero values, which incorrectly blocked a legitimate `0` deal value in some cases while allowing negative numbers through).
- `getMonthKey()` now derives the year/month from local date parts (`getFullYear()`/`getMonth()`) instead of `toISOString().slice(0,7)`, avoiding a day/month misattribution for users near the UTC day boundary.
- Monthly revenue and "won this month" stat calculations now key off `wonAt` (falling back to `updatedAt`/`createdAt`) instead of `updatedAt`, so editing a won deal's notes later no longer shifts which month its revenue is counted in.
- Supabase sync no longer permanently disables itself after one failed request (previously a single error set a `usingSupabase = false` flag for the rest of the session); it now retries on every subsequent save via a per-attempt `try/catch`.

### Fixed
- Restoring an entry from Trash (`undoTrash`) now strips the `deletedAt` field before re-inserting it into the active pipeline, so a restored entry is no longer mistaken for a trashed one.

## [0.1.0] - 2026-07-23

Initial commit (`44cc131`, "Initial marketing dashboard"). First working version of the dashboard.

### Added
- Single-file HTML/CSS/JS marketing/sales pipeline dashboard with no build step.
- Prospect/client entry form (add/edit/delete) covering company, contact person, phone, email, deal value, status, notes, last-contact date, next-follow-up date, and logged-by.
- Shared pipeline table with search, status filter, and sort (by next follow-up date, company, status, or deal value).
- Stat tiles for deals won this month, win rate, overdue items, lead target, revenue target, and open deals.
- Editable monthly revenue and lead targets with progress bars.
- Due-today/this-week list with overdue/soon flagging.
- Trash with a 5-entry cap and undo.
- Activity feed logging adds/edits/deletes/restores (last 50).
- Month-over-month history archiving (last 6 months) triggered automatically on load.
- One-click WhatsApp and email outreach links generated per pipeline entry.
- `localStorage` persistence under `marketing-dashboard-state-v1`.
- Optional Supabase sync (`dashboard_state` table) via a CDN-loaded `@supabase/supabase-js` client, with graceful fallback to local-only storage if unconfigured or unreachable.
