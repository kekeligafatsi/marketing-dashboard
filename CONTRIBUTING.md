# Contributing

Thanks for helping maintain the Marketing Dashboard. This is a single-file, no-build-step project, so the workflow is intentionally lightweight.

## Local setup

There is no package manager, no dependencies to install, and no build step.

1. Clone the repo: `git clone https://github.com/kekeligafatsi/marketing-dashboard.git`
2. Open `index.html` directly in a browser (double-click it, or `File > Open`), **or** serve it with any static file server if you want it on `http://localhost` instead of `file://` (for example `python -m http.server` from the project folder, then visit the printed URL). Either way works identically — the app only needs a browser that can run `localStorage` and load the Supabase CDN script.
3. If you want to test the shared-sync path against your **own** data instead of the team's live Supabase project, temporarily edit the `window.MARKETING_DASHBOARD_CONFIG` object near the top of the `<script>` block in `index.html` with your own project's URL/anon key and a `dashboard_state` table (`key` text, `value` jsonb, `updated_at` timestamptz) — see [README.md](README.md#setup--run-instructions). Do not commit your personal credentials over the team's config.

## Making changes

- Everything lives in `index.html` — markup, CSS, and JS in one file. There's no separate components/services/styles directory to navigate.
- Keep the file dependency-free: don't introduce a bundler, framework, or additional CDN scripts unless there's a strong reason, since zero-install simplicity is a deliberate design choice (see [TECHSTACK.md](TECHSTACK.md#notable-architectural-decisions)).
- Follow the existing patterns already in the file: mutate `state`, then call `saveState()`; escape any user-supplied string with `escapeHtml()` before interpolating it into rendered HTML; validate any new persisted field in `sanitizeEntry`/`sanitizeState` so malformed local/remote data can't crash rendering.

## Testing

There is no automated test suite, linter, or CI configuration in this project — verify changes manually in a browser:

- Add, edit, and soft-delete a pipeline entry; confirm it persists after a page reload (`localStorage`).
- If Supabase is configured, confirm the sync banner (`#syncStatus`) behaves correctly — hidden on success, shown on a forced failure (e.g. temporarily point the config at an invalid URL).
- Check the responsive layout below the `980px` breakpoint (columns collapse, two table columns hide).
- Check the browser console for errors/warnings during load, save, and sync.

## Commit conventions

- This repo's history is small; there isn't an established strict commit-message format yet. When your change is user-visible, add an entry to the matching [Keep a Changelog](https://keepachangelog.com/) section (`Added` / `Changed` / `Fixed` / `Removed` / `Deprecated` / `Security`) under `[Unreleased]` in [CHANGELOG.md](CHANGELOG.md), the same way prior changes are documented there.
- Write commit messages that explain *why*, not just *what*, especially for anything touching the sync/concurrency logic (`persistToSupabase`, `sanitizeState`) or data shape (`DEFAULT_STATE`, `Entry`).

## Pull request process

1. Create a branch off `master` for your change.
2. Push your branch and open a PR against `kekeligafatsi/marketing-dashboard` (`gh pr create` or the GitHub UI).
3. In the PR description, note what you tested manually (there's no CI to lean on).
4. Since the whole app is one file, expect diffs to be easy to review in full — please avoid unrelated reformatting in the same PR as a functional change, to keep diffs reviewable.
