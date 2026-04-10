# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A two-file HTML web app for tracking dental school clinical graduation requirements. No build step, no package manager. Data stored in Supabase (hosted Postgres). Hosted on GitHub Pages.

## Files

- `login.html` — email/password login and signup. Redirects to `index.html` on auth success.
- `index.html` — the tracker app. Requires a valid Supabase session; redirects to `login.html` if none.

Both files load Supabase JS from CDN (`@supabase/supabase-js@2`). The Supabase anon key is embedded in both files — this is intentional and safe; RLS enforces per-user data isolation.

## Running the app

```bash
open login.html
```

Or open via GitHub Pages URL. No local server needed.

## Supabase project

- URL: `https://aaowuzjcdrkifmejjmzc.supabase.co`
- Tables: `entries`, `doctors`, `requirements` — all with RLS (`auth.uid() = user_id`)
- Email confirmation: should be disabled in Supabase dashboard → Authentication → Configuration

## Architecture (index.html)

The `<script>` block is organized in this order:

1. **Supabase client** — `const sb = createClient(URL, KEY)` at the very top
2. **Constants** — `DISCIPLINES` array, `DEFAULT_REQUIREMENTS` object (8 disciplines, ~57 procedures, each with `{ formative: N, summative: N }`)
3. **State layer** — module-level `let state = { requirements, entries, doctors }`. `getState()` is a thin wrapper returning `state`. `loadFromSupabase()` fetches all three tables in parallel on init. `seedRequirements(userId)` inserts default requirements for new users.
4. **Query helpers** — `getEntriesFor()`, `getCountFor()`, `computeGlobalProgress()`, `computeDiscProgress()`. Progress functions cap counts at the requirement ceiling with `Math.min`.
5. **ID helpers** — `sanitizeId()`, `panelId()`, `rowId()`. Every procedure row and log panel gets a deterministic DOM ID. IDs are of the form `row-{disc}-{proc}-{type}` with special characters replaced by `_`.
6. **Log panel** — `renderLogPanel()`, `toggleLogPanel()`, `deleteEntry()`. `toggleLogPanel` closes all open panels before opening the clicked one. `deleteEntry` is async (Supabase delete), saves open row context before `render()` so it can reopen the panel afterward.
7. **Discipline view** — `statusClass()`, `buildReqRows()`, `renderDiscipline()`. Procedures with `needed === 0` for a given type are omitted from that section.
8. **Modal** — `openModal()`, `closeModal()`, `populateDoctorSelect()`, `saveEntry()`. `saveEntry()` is async; inserts entry (and optionally doctor) to Supabase, reconstructs entry from DB response, updates local state. `modalContext` holds `{ discipline, proc, type }` while modal is open.
9. **Settings** — `renderSettings()`, `renderReqSettings()`, `renderDoctorsSettings()`, `exportBackup()`. All mutations are async Supabase calls.
10. **Nav + sidebar** — `renderNavCounts()`, `renderSidebar()`, `render()`, `showView()`.
11. **Event listeners + async init IIFE** — init checks session, calls `loadFromSupabase()`, then `render()`.

## State shape (in-memory)

```js
{
  requirements: { [discipline]: { [procedure]: { formative: N, summative: N } } },
  entries: [{ id, discipline, procedure, type, date, doctor, axiUmNumber, axiUmFormCode, notes, createdAt }],
  doctors: [string]
}
```

DB column names are snake_case (`axium_number`, `axium_form_code`). JS uses camelCase. The mapping happens in `loadFromSupabase()` and `saveEntry()`.

## XSS

All user-supplied data must be passed through `esc()` before interpolating into `innerHTML`. The `esc()` helper escapes `&`, `<`, `>`, `"`. Using `.textContent` is always safe without `esc()`.

## Key design decisions

- **No frameworks** — vanilla JS only. Supabase JS is the only external dependency (CDN).
- **Local state cache** — all data fetched once on load into `state`. Mutations update Supabase first, then update `state` in place and call `render()`. No optimistic updates.
- **Full re-render on every state change** — `render()` rebuilds sidebar + discipline view from scratch. Fast enough for this data size.
- **Settings are lazy** — `renderSettings()` is only called when the Settings nav tab is opened.
- `DEFAULT_REQUIREMENTS` must not be mutated; first-login seeding deep-clones it into the DB.

## Docs

- Design spec (original): `docs/superpowers/specs/2026-04-10-swipe-tracker-design.md`
- Design spec (Supabase): `docs/superpowers/specs/2026-04-10-swipe-tracker-supabase.md`
- Implementation plan (original): `docs/superpowers/plans/2026-04-10-swipe-tracker.md`
- Implementation plan (Supabase): `docs/superpowers/plans/2026-04-10-swipe-tracker-supabase.md`
