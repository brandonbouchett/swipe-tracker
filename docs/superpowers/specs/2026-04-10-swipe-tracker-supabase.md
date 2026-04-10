# Swipe Tracker — Supabase Migration Design Spec
**Date:** 2026-04-10

---

## Context

The existing Swipe Tracker is a single `index.html` file using `localStorage` for persistence. The goal is to migrate data storage to Supabase (hosted Postgres), add email/password authentication, and make the app hostable on GitHub Pages so multiple dental students can each maintain their own private records.

---

## Decisions Made

| Question | Decision |
|---|---|
| Database | Supabase (hosted Postgres + Auth) |
| Auth method | Email + password (Supabase Auth, no email confirmation required) |
| File structure | Two HTML files: `login.html` + `index.html` |
| Hosting | GitHub Pages |
| Data privacy | Full row-level security — each user sees only their own data |
| Client library | Supabase JS via CDN — no build step |
| Local caching | Fetch all data on load into local `state`; mutations update Supabase then re-render |

---

## Architecture

**Two static HTML files served from GitHub Pages:**

- `login.html` — signup/login page. Dark theme matching the app. Handles both new account creation and returning logins via toggle. On success, redirects to `index.html`.
- `index.html` — the tracker app (existing app, modified). Checks for a valid Supabase session on load; redirects to `login.html` if none. Adds a logout button to the nav bar.

Both files load the Supabase JS client from CDN:
```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

The Supabase anon key is embedded directly in the HTML. This is safe and intentional — RLS on the database is what protects user data, not key secrecy.

---

## Database Schema

### `entries`
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` | Primary key, `gen_random_uuid()` default |
| `user_id` | `uuid` | References `auth.users`, not null |
| `discipline` | `text` | e.g. `"Diagnosis"` |
| `procedure` | `text` | e.g. `"NPI"` |
| `type` | `text` | `"formative"` or `"summative"` |
| `date` | `date` | ISO date string |
| `doctor` | `text` | |
| `axium_number` | `text` | |
| `axium_form_code` | `text` | |
| `notes` | `text` | |
| `created_at` | `timestamptz` | `now()` default |

### `doctors`
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` | Primary key, `gen_random_uuid()` default |
| `user_id` | `uuid` | References `auth.users`, not null |
| `name` | `text` | |

### `requirements`
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` | Primary key, `gen_random_uuid()` default |
| `user_id` | `uuid` | References `auth.users`, not null |
| `discipline` | `text` | |
| `procedure` | `text` | |
| `formative` | `integer` | |
| `summative` | `integer` | |

### Row Level Security

All three tables have RLS enabled. Each table gets two policies:

```sql
-- SELECT
CREATE POLICY "Users can read own rows"
  ON <table> FOR SELECT
  USING (auth.uid() = user_id);

-- INSERT / UPDATE / DELETE
CREATE POLICY "Users can write own rows"
  ON <table> FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);
```

---

## Auth Flow

1. User visits `login.html`
2. Toggles between "Log in" and "Sign up" modes via a button
3. **Sign up:** creates account with email + password. Email confirmation is disabled in Supabase project settings — user is authenticated immediately.
4. **Log in:** authenticates with email + password.
5. On success: redirect to `index.html`
6. `index.html` on load: calls `supabase.auth.getSession()` → if no session, redirect to `login.html`
7. Logout button in nav bar: calls `supabase.auth.signOut()` → redirect to `login.html`

---

## Data Layer (index.html changes)

### Local state cache

On page load, fetch all three tables into a local `state` object that mirrors the existing shape:

```js
let state = {
  requirements: {},  // { [discipline]: { [procedure]: { formative: N, summative: N } } }
  entries: [],       // array of entry objects
  doctors: []        // array of name strings
};
```

The UI render functions remain synchronous — they read from `state` exactly as before.

### Initial load sequence

```
1. getSession() → no session → redirect to login.html
2. getSession() → session found → fetch all three tables in parallel
3. Transform DB rows into state shape
4. If requirements table is empty for this user → seed from DEFAULT_REQUIREMENTS
5. Render app
6. Hide loading overlay
```

### Mutations

Every mutation follows this pattern:
1. Call Supabase (insert/update/delete)
2. On success: update local `state`
3. Call `render()` to refresh UI

No optimistic updates — wait for Supabase to confirm before re-rendering. For this data size (tens of entries), the latency is imperceptible.

### Function mapping

| Old function | New behavior |
|---|---|
| `getState()` | Returns local `state` (still synchronous) |
| `setState(data)` | Removed — mutations go directly to Supabase |
| `saveEntry()` | `supabase.from('entries').insert(...)` → update `state.entries` |
| `deleteEntry()` | `supabase.from('entries').delete().eq('id', id)` → update `state.entries` |
| `addDoctor()` | `supabase.from('doctors').insert(...)` → update `state.doctors` |
| `deleteDoctor()` | `supabase.from('doctors').delete().eq('id', id)` → update `state.doctors` |
| Requirements edit | `supabase.from('requirements').update(...)` → update `state.requirements` |

### Loading overlay

A full-screen dark overlay with a "Loading…" message shown during the initial async fetch. Removed once the app is ready to render.

```html
<div id="loading-overlay">Loading…</div>
```

---

## login.html Design

- Dark background matching the app (`#0f0f1a`)
- Centered card with:
  - App name ("Swipe Tracker") at top
  - Email + password inputs
  - Primary action button ("Log in" or "Create account")
  - Toggle link: "Don't have an account? Sign up" / "Already have an account? Log in"
  - Error message area for auth failures (wrong password, email already taken, etc.)
- No external CSS — inline styles matching app theme

---

## Supabase Project Setup

1. Create project via Supabase MCP
2. Run migration SQL to create tables and RLS policies
3. Disable email confirmation in Auth settings (so users can sign in immediately after signup)
4. Copy project URL and anon key into both HTML files

---

## GitHub Pages Hosting

1. Create GitHub repository (public or private — GitHub Pages works on both with the right plan)
2. Push `login.html` and `index.html` to `main` branch
3. Enable GitHub Pages in repo Settings → Pages → Source: `main` branch, root `/`
4. App is live at `https://<username>.github.io/<repo-name>/login.html`

---

## Error Handling

- Auth errors (wrong password, email taken): display inline in `login.html` error area
- Supabase fetch errors on load: show "Failed to load data. Please refresh." message
- Supabase mutation errors: `alert()` with brief message (same pattern as existing app)
- Session expiry: Supabase JS handles token refresh automatically. If refresh fails, `onAuthStateChange` fires with `SIGNED_OUT` → redirect to `login.html`

---

## Out of Scope

- Password reset / "forgot password" flow (can add later)
- Email confirmation (disabled for simplicity)
- Admin/instructor view of all students
- Offline mode / sync conflict resolution
- Migrating existing localStorage data to Supabase
