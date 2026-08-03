# Project Tree

This project is a **single self-contained HTML file** — there is no folder structure, no components directory, no services directory, and no build output to speak of. The tree below is therefore trivial at the filesystem level; the more useful "architecture map" is the annotated breakdown of `index.html`'s internal sections further down.

```
Marketing dashboard/
├── .git/              # Git repository metadata (1 commit on record); not enumerated further
├── index.html         # The entire application: markup, CSS, and JS in one file
├── README.md          # Human-facing overview (generated)
├── TECHSTACK.md       # Technical stack detail (generated)
├── CHANGELOG.md       # Version history (generated)
├── TREEVIEW.md        # This file (generated)
├── ARCHITECTURE.md    # Data-flow/consistency model between the page, localStorage, and Supabase (generated)
└── CONTRIBUTING.md    # Local setup, manual test checklist, PR process (generated)
```

No `node_modules`, lockfiles, `dist`/`build` output, or config files (`package.json`, `tsconfig.json`, bundler configs) exist in this project — consistent with it being a zero-dependency static HTML file.

Not generated for this project: `API.md` (no backend routes — Supabase is used only as a key-value store, not a REST API this app exposes), `COMPONENTS.md` (no component framework or reusable parameterized UI components — the single-purpose DOM sections are already catalogued below), and `.env.example` (the app reads no environment variables at all; its one piece of config, the Supabase URL/anon key, is a hardcoded `window.MARKETING_DASHBOARD_CONFIG` object inline in `index.html`, not sourced from `.env`/`process.env`).

## Inside `index.html` — annotated map

Since everything lives in one file, this is the real architecture map: the file's logical regions in the order they appear.

### `<head>` / `<style>` (config, no folders)
| Region | Purpose |
|---|---|
| `:root` CSS variables | Color palette (navy/blue dark theme, status colors: danger/warning/success) shared by every rule below it |
| Layout rules (`.shell`, `.grid`, `.layout`, `.target-row`, `.toolbar`, `.form-grid`) | Grid/flex layout for the page shell and each major section |
| Component-style rules (`.card`, `.stat-card`, `.due-item`, `.trash-item`, `.status-badge`, `.pill`, `.modal`, `.toast`) | Reusable visual "components" expressed purely as CSS classes, applied to plain `<div>`/`<table>` markup |
| Responsive breakpoint (`@media max-width: 980px`) | Collapses multi-column grids to one column and hides two table columns on small screens |

### `<body>` — UI sections (the "blocks")
| Section (element id) | Renders | Populated by |
|---|---|---|
| `.topbar` | Title, subtitle, "New Prospect" and "Refresh View" buttons | Static markup + `newEntryBtn`/`refreshBtn` listeners |
| `#syncStatus` | Warning banner shown only when a Supabase sync attempt has failed | `updateSyncBanner()` |
| `#statsGrid` | 6 stat cards: won this month, win rate, overdue items, lead target, revenue target, open deals | `renderStats()` |
| Target row (`#revenueTargetInput`/`#revenueProgress`, `#leadTargetInput`/`#leadProgress`) | Editable monthly revenue/lead targets with progress bars | `renderProgress()`; inputs write to `state` on `change` |
| `#dueList` | Entries overdue or due within 7 days, flagged `flag-overdue`/`flag-soon` | `renderDueList()` / `getDueItems()` |
| `#trashList` | Soft-deleted entries with an Undo button each | `renderTrash()` / `undoTrash()` |
| Pipeline card (`#searchInput`, `#statusFilter`, `#sortField`, `#clearFiltersBtn`, `#pipelineTableBody`) | Searchable/filterable/sortable table of all entries, with Edit/Delete/WhatsApp/Email actions per row | `renderTable()` |
| `#activityFeed` | Chronological log of the last 50 add/edit/delete/restore actions | `renderActivity()` |
| `#historyList` | Archived month-over-month target vs. actual snapshots (last 6 months) | `renderHistory()` / `archiveMonthIfNeeded()` |
| Quick Notes card | Static explanatory text (no-login, live-shared pipeline note) | Static markup |
| `#entryModal` / `#entryForm` | Add/Edit Prospect modal form (company, contact, phone, email, deal value, status, dates, logged-by, notes) | `openModal()` / `addOrUpdateEntry()` |
| `#toast` | Ephemeral bottom-right notification (e.g. "Entry saved successfully.") | `showToast()` |

### `<script>` — data shape and logic (the "types" and "services" for a plain-JS app)

**Core state shape** (`DEFAULT_STATE`, persisted whole):
```
{
  revenueTarget: number,
  leadTarget: number,
  entries: Entry[],
  activity: ActivityItem[],
  trash: Entry[],       // soft-deleted entries, each with an added deletedAt
  history: HistoryItem[]
}
```

**`Entry` shape** (one pipeline row), enforced by `sanitizeEntry`:
```
{
  id, company, contactPerson, phone, email,
  dealValue: number (>= 0),
  status: 'contacted' | 'quoted' | 'follow-up-due' | 'won' | 'lost',
  notes, lastContactDate, nextFollowUpDate, loggedBy,
  createdAt, updatedAt,
  wonAt?: string,       // set when status transitions to 'won'
  deletedAt?: string    // present only while sitting in trash
}
```

**`ActivityItem`**: `{ id, text, createdAt }` — capped at 50, newest first.

**`HistoryItem`**: `{ monthKey, monthLabel, revenueTarget, revenueActual, leadTarget, leadActual, archivedAt }` — capped at 6, one per month, appended by `archiveMonthIfNeeded()`.

**Function groups** (no separate files — all in the one `<script>` block):
- **Persistence / sync** — `loadState`, `saveState`, `initStorage`, `persistToSupabase`, `updateSyncBanner`: read/write `localStorage`, and optionally read/write the Supabase `dashboard_state` table with an optimistic-concurrency check against `updated_at`.
- **Validation** — `sanitizeEntry`, `sanitizeState`, `escapeHtml`, `statusClass`: coerce untrusted JSON into the expected shape and escape values before they hit the DOM.
- **Derived data / calculations** — `getMonthKey`, `formatMonthLabel`, `sumWonRevenueForMonth`, `countEntriesByMonth`, `getDueItems`, `formatMoney`: pure functions computing stats, due-item flags, and currency formatting (GH₵ via `Intl.NumberFormat`) from `state`.
- **Rendering** — `render`, `renderStats`, `renderProgress`, `renderDueList`, `renderTrash`, `renderActivity`, `renderHistory`, `renderTable`: each rewrites one section's `innerHTML` from current `state` and re-binds that section's event listeners.
- **Mutations** — `addOrUpdateEntry`, `removeEntry`, `undoTrash`, plus the target-input `change` handlers: the only places `state` is mutated; each ends by calling `saveState()`.
- **Modal control** — `openModal`, `closeModal`: populate/clear the add-edit form and toggle the modal's visibility.
- **Outreach helpers** — `buildWhatsAppLink`, `buildEmailLink`: build a `wa.me` URL and a `mailto:` URL pre-filled with a follow-up message per entry.
- **Misc UI** — `showToast`: transient bottom-right toast notifications.

**External service integration**: a single `<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2">` tag loads the Supabase client; `window.MARKETING_DASHBOARD_CONFIG` (defined inline, just below) supplies the project URL and anon key used by `initStorage()`/`persistToSupabase()`.
