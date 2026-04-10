# Swipe Tracker — Supabase Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate Swipe Tracker from localStorage to Supabase (Postgres + Auth), add email/password login with per-user private data, and make it hostable on GitHub Pages.

**Architecture:** Two static HTML files (`login.html` + `index.html`) served from GitHub Pages. Both load the Supabase JS client from CDN — no build step. Supabase handles auth (email/password) and three Postgres tables (`entries`, `doctors`, `requirements`) with Row Level Security enforcing per-user data isolation. `index.html` caches all data in a module-level `state` object on load; mutations update Supabase then update local state and re-render.

**Tech Stack:** Vanilla JS, Supabase JS v2 (CDN), Supabase hosted Postgres + Auth, GitHub Pages

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `login.html` | Create | Login + signup page, dark theme matching app, redirects to `index.html` on auth success |
| `index.html` | Modify | Tracker app — add Supabase client, loading overlay, session check, logout; replace localStorage state layer with Supabase calls |

**After Task 1 completes:** Replace `SUPABASE_URL` and `SUPABASE_ANON_KEY` placeholders throughout this plan with the real values from the Supabase project.

---

## Task 1: Create Supabase project and apply schema

**Files:**
- No file changes — this task uses Supabase MCP tools only

- [ ] **Step 1: List Supabase organizations to get org ID**

Use the `mcp__claude_ai_Supabase__list_organizations` tool. Note the `id` of the organization to use.

- [ ] **Step 2: Create Supabase project**

Use `mcp__claude_ai_Supabase__create_project` with:
- `name`: `swipe-tracker`
- `organization_id`: (from Step 1)
- `region`: `us-east-1`
- `confirm_cost_id`: use `mcp__claude_ai_Supabase__get_cost` first if needed, then `mcp__claude_ai_Supabase__confirm_cost`

Wait for the project to be ready (status `ACTIVE_HEALTHY`). Note the project `id`.

- [ ] **Step 3: Get project URL and anon key**

Use `mcp__claude_ai_Supabase__get_project_url` and `mcp__claude_ai_Supabase__get_publishable_keys` with the project ID from Step 2.

Note down:
- `SUPABASE_URL` — looks like `https://xxxxxxxxxxxx.supabase.co`
- `SUPABASE_ANON_KEY` — long JWT string

- [ ] **Step 4: Apply database schema migration**

Use `mcp__claude_ai_Supabase__apply_migration` with the project ID and this SQL:

```sql
-- entries table
CREATE TABLE entries (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES auth.users NOT NULL,
  discipline text NOT NULL,
  procedure text NOT NULL,
  type text NOT NULL CHECK (type IN ('formative', 'summative')),
  date date NOT NULL,
  doctor text NOT NULL,
  axium_number text NOT NULL DEFAULT '',
  axium_form_code text NOT NULL DEFAULT '',
  notes text NOT NULL DEFAULT '',
  created_at timestamptz NOT NULL DEFAULT now()
);

-- doctors table
CREATE TABLE doctors (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES auth.users NOT NULL,
  name text NOT NULL
);

-- requirements table
CREATE TABLE requirements (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES auth.users NOT NULL,
  discipline text NOT NULL,
  procedure text NOT NULL,
  formative integer NOT NULL DEFAULT 0,
  summative integer NOT NULL DEFAULT 0
);

-- Enable RLS
ALTER TABLE entries ENABLE ROW LEVEL SECURITY;
ALTER TABLE doctors ENABLE ROW LEVEL SECURITY;
ALTER TABLE requirements ENABLE ROW LEVEL SECURITY;

-- RLS policies: each user can only read/write their own rows
CREATE POLICY "entries_own" ON entries
  FOR ALL USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);

CREATE POLICY "doctors_own" ON doctors
  FOR ALL USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);

CREATE POLICY "requirements_own" ON requirements
  FOR ALL USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);
```

- [ ] **Step 5: Disable email confirmation**

Use `mcp__claude_ai_Supabase__execute_sql` with the project ID to run:

```sql
UPDATE auth.config SET value = 'false' WHERE key = 'MAILER_AUTOCONFIRM';
```

If that returns an error (auth config is not always accessible via SQL), note it — the Supabase dashboard Auth settings page has a toggle "Enable email confirmations" that must be turned OFF. Include a note in the commit message so the user knows to check this manually if needed.

- [ ] **Step 6: Verify tables exist**

Use `mcp__claude_ai_Supabase__list_tables` with the project ID. Confirm `entries`, `doctors`, and `requirements` appear.

- [ ] **Step 7: Commit**

```bash
git add docs/
git commit -m "chore: provision Supabase project and apply schema

Tables: entries, doctors, requirements
RLS enabled — per-user data isolation
Project: swipe-tracker"
```

---

## Task 2: Create login.html

**Files:**
- Create: `login.html`

- [ ] **Step 1: Create login.html**

Create the file at the repo root. Replace `SUPABASE_URL` and `SUPABASE_ANON_KEY` with the real values from Task 1.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Swipe Tracker — Login</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    :root {
      --bg: #0f0f1a; --surface: #1a1a2e; --border: #2a2a4a;
      --accent: #4a90d9; --accent-lt: #6a9fd8;
      --text: #e0e0e0; --muted: #888; --red: #f44336;
    }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: var(--bg); color: var(--text);
      min-height: 100vh; display: flex; align-items: center; justify-content: center;
    }
    .card {
      background: var(--surface); border: 1px solid var(--border);
      border-radius: 12px; padding: 36px 32px; width: 360px;
    }
    .logo {
      font-size: 22px; font-weight: 700; color: var(--accent-lt);
      letter-spacing: 0.5px; text-align: center; margin-bottom: 6px;
    }
    .tagline {
      font-size: 13px; color: var(--muted); text-align: center; margin-bottom: 28px;
    }
    .form-row { margin-bottom: 14px; }
    .form-row label {
      display: block; font-size: 11px; color: var(--muted);
      text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 5px;
    }
    .form-row input {
      width: 100%; background: #0f0f1a; border: 1px solid var(--border);
      border-radius: 6px; padding: 9px 12px; color: var(--text);
      font-size: 14px; outline: none; transition: border-color 0.15s;
    }
    .form-row input:focus { border-color: var(--accent); }
    .btn {
      width: 100%; padding: 10px; border-radius: 6px; font-size: 14px;
      font-weight: 600; cursor: pointer; border: none; margin-top: 4px;
      transition: opacity 0.15s;
    }
    .btn:disabled { opacity: 0.5; cursor: not-allowed; }
    .btn-primary { background: #2a5aaa; color: white; }
    .btn-primary:not(:disabled):hover { background: #3369c4; }
    #error-msg {
      font-size: 12px; color: var(--red); margin-top: 12px;
      text-align: center; min-height: 16px;
    }
    .toggle-row {
      text-align: center; margin-top: 20px; font-size: 13px; color: var(--muted);
    }
    .toggle-link {
      color: var(--accent-lt); cursor: pointer; text-decoration: underline;
    }
  </style>
</head>
<body>
  <div class="card">
    <div class="logo">Swipe Tracker</div>
    <div class="tagline">Dental clinical requirements tracker</div>

    <div class="form-row">
      <label for="email">Email</label>
      <input type="email" id="email" placeholder="you@example.com" autocomplete="email">
    </div>
    <div class="form-row">
      <label for="password">Password</label>
      <input type="password" id="password" placeholder="••••••••" autocomplete="current-password">
    </div>

    <button class="btn btn-primary" id="submit-btn">Log in</button>
    <div id="error-msg"></div>

    <div class="toggle-row">
      <span id="toggle-text">Don't have an account?</span>
      <span class="toggle-link" id="toggle-link">Sign up</span>
    </div>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <script>
    const { createClient } = supabase;
    const sb = createClient('SUPABASE_URL', 'SUPABASE_ANON_KEY');

    let mode = 'login'; // 'login' | 'signup'

    // If already logged in, go straight to the app
    sb.auth.getSession().then(({ data }) => {
      if (data.session) window.location.href = 'index.html';
    });

    document.getElementById('toggle-link').addEventListener('click', () => {
      mode = mode === 'login' ? 'signup' : 'login';
      const isLogin = mode === 'login';
      document.getElementById('submit-btn').textContent = isLogin ? 'Log in' : 'Create account';
      document.getElementById('toggle-text').textContent = isLogin
        ? "Don't have an account?" : 'Already have an account?';
      document.getElementById('toggle-link').textContent = isLogin ? 'Sign up' : 'Log in';
      document.getElementById('error-msg').textContent = '';
      document.getElementById('password').autocomplete = isLogin ? 'current-password' : 'new-password';
    });

    async function submit() {
      const email = document.getElementById('email').value.trim();
      const password = document.getElementById('password').value;
      const errEl = document.getElementById('error-msg');
      const btn = document.getElementById('submit-btn');

      errEl.textContent = '';
      if (!email || !password) {
        errEl.textContent = 'Please enter your email and password.';
        return;
      }

      btn.disabled = true;
      btn.textContent = mode === 'login' ? 'Logging in…' : 'Creating account…';

      const { error } = mode === 'login'
        ? await sb.auth.signInWithPassword({ email, password })
        : await sb.auth.signUp({ email, password });

      if (error) {
        errEl.textContent = error.message;
        btn.disabled = false;
        btn.textContent = mode === 'login' ? 'Log in' : 'Create account';
        return;
      }

      window.location.href = 'index.html';
    }

    document.getElementById('submit-btn').addEventListener('click', submit);
    document.getElementById('password').addEventListener('keydown', e => {
      if (e.key === 'Enter') submit();
    });
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify in browser**

Open `login.html` directly in a browser. Confirm:
- Page renders with dark background, card, email/password fields
- "Sign up" toggle switches button text and toggle label
- Clicking submit with empty fields shows error: "Please enter your email and password."

- [ ] **Step 3: Commit**

```bash
git add login.html
git commit -m "feat: add login/signup page with Supabase auth"
```

---

## Task 3: Add Supabase client + auth shell to index.html

**Files:**
- Modify: `index.html`

This task adds the Supabase CDN script, loading overlay HTML, session guard on init, and logout button to the nav. No data calls yet — the app will still crash on load after this task (state layer isn't wired yet). That's expected.

- [ ] **Step 1: Add Supabase CDN script to `<head>`**

In `index.html`, find the closing `</style>` tag (around line 260) and add immediately after:

```html
  </style>
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

- [ ] **Step 2: Add loading overlay to `<body>`**

Find the line `<div id="app">` (the main app layout wrapper) and add this block immediately before it:

```html
  <!-- Loading overlay -->
  <div id="loading-overlay" style="
    position:fixed; inset:0; background:#0f0f1a;
    display:flex; align-items:center; justify-content:center;
    font-size:15px; color:#888; z-index:9999;
  ">Loading…</div>
```

- [ ] **Step 3: Add logout button to the nav HTML**

Find the nav HTML section. The nav currently has `.logo`, `.nav-links`, and `#global-counts`. Add a logout button after `#global-counts`:

```html
      <div id="global-counts"></div>
      <button id="logout-btn" style="
        padding:6px 12px; border-radius:6px; font-size:12px;
        background:none; border:1px solid #2a2a4a; color:#888; cursor:pointer;
      ">Log out</button>
```

- [ ] **Step 4: Initialize Supabase client at the top of the `<script>` block**

Find the `<script>` tag (around line 363) and add these two lines immediately after it, before the `// CONSTANTS` comment:

```js
  <script>
    const { createClient } = supabase;
    const sb = createClient('SUPABASE_URL', 'SUPABASE_ANON_KEY');
```

Replace `SUPABASE_URL` and `SUPABASE_ANON_KEY` with the real values from Task 1.

- [ ] **Step 5: Replace the init IIFE with an async session-guarded init**

Find the init IIFE at the bottom of the script block (around line 955):

```js
    (function init() {
      try {
        if (!localStorage.getItem(STORAGE_KEY)) setState(buildDefaultState());
      } catch (_) {
        storageAvailable = false;
      }
      if (!storageAvailable) {
        document.getElementById('storage-warning').style.display = 'block';
      }
      render();
    })();
```

Replace it with:

```js
    document.getElementById('logout-btn').addEventListener('click', async () => {
      await sb.auth.signOut();
      window.location.href = 'login.html';
    });

    (async function init() {
      const { data: { session } } = await sb.auth.getSession();
      if (!session) {
        window.location.href = 'login.html';
        return;
      }
      // Data load will be wired in Task 4
      document.getElementById('loading-overlay').style.display = 'none';
      render();
    })();
```

- [ ] **Step 6: Verify**

Open `index.html` directly (not through a server — just open the file). Because there's no active session, the page should immediately redirect to `login.html`. If it does, the session guard is working.

Then open `login.html`, create an account, and verify you land on `index.html` (it will likely show the app with whatever localStorage data was there before — that's fine, state layer migration is Task 4).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add Supabase client, loading overlay, session guard, logout to index.html"
```

---

## Task 4: Replace state layer with Supabase data load

**Files:**
- Modify: `index.html` (state layer section, ~lines 446–489; init IIFE)

This task replaces localStorage with a module-level `state` variable populated from Supabase on login. After this task, the app loads real data from the database.

- [ ] **Step 1: Replace the localStorage state layer**

Find this entire block (lines ~446–489):

```js
    // ─────────────────────────────────────────────
    // STATE (localStorage)
    // ─────────────────────────────────────────────
    let storageAvailable = true;

    function getState() {
      try {
        const raw = localStorage.getItem(STORAGE_KEY);
        if (!raw) return buildDefaultState();
        return JSON.parse(raw);
      } catch (e) {
        if (e instanceof SyntaxError) {
          console.warn('swipeTracker: stored data unreadable, resetting.', e);
          return buildDefaultState();
        }
        storageAvailable = false;
        return buildDefaultState();
      }
    }

    function setState(data) {
      try {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
      } catch (e) {
        storageAvailable = false;
        document.getElementById('storage-warning').style.display = 'block';
      }
    }

    function buildDefaultState() {
      return {
        requirements: JSON.parse(JSON.stringify(DEFAULT_REQUIREMENTS)),
        entries: [],
        doctors: [],
      };
    }

    function generateId() {
      return crypto.randomUUID();
    }
```

Replace with:

```js
    // ─────────────────────────────────────────────
    // STATE (Supabase)
    // ─────────────────────────────────────────────
    let state = { requirements: {}, entries: [], doctors: [] };

    // All existing render functions call getState() — keep it as a thin wrapper
    function getState() { return state; }

    async function loadFromSupabase() {
      const [entriesRes, doctorsRes, reqsRes] = await Promise.all([
        sb.from('entries').select('*').order('created_at', { ascending: true }),
        sb.from('doctors').select('*').order('name'),
        sb.from('requirements').select('*'),
      ]);

      if (entriesRes.error) throw entriesRes.error;
      if (doctorsRes.error) throw doctorsRes.error;
      if (reqsRes.error) throw reqsRes.error;

      state.entries = entriesRes.data.map(row => ({
        id: row.id,
        discipline: row.discipline,
        procedure: row.procedure,
        type: row.type,
        date: row.date,
        doctor: row.doctor,
        axiUmNumber: row.axium_number || '',
        axiUmFormCode: row.axium_form_code || '',
        notes: row.notes || '',
        createdAt: new Date(row.created_at).getTime(),
      }));

      state.doctors = doctorsRes.data.map(row => row.name);

      if (reqsRes.data.length === 0) {
        await seedRequirements();
      } else {
        state.requirements = {};
        for (const row of reqsRes.data) {
          if (!state.requirements[row.discipline]) state.requirements[row.discipline] = {};
          state.requirements[row.discipline][row.procedure] = {
            formative: row.formative,
            summative: row.summative,
          };
        }
      }
    }

    async function seedRequirements() {
      const { data: { user } } = await sb.auth.getUser();
      const rows = [];
      for (const [discipline, procs] of Object.entries(DEFAULT_REQUIREMENTS)) {
        for (const [procedure, req] of Object.entries(procs)) {
          rows.push({
            user_id: user.id,
            discipline,
            procedure,
            formative: req.formative,
            summative: req.summative,
          });
        }
      }
      const { error } = await sb.from('requirements').insert(rows);
      if (error) throw error;
      state.requirements = JSON.parse(JSON.stringify(DEFAULT_REQUIREMENTS));
    }
```

- [ ] **Step 2: Update the init IIFE to call loadFromSupabase()**

Find the init IIFE added in Task 3:

```js
    (async function init() {
      const { data: { session } } = await sb.auth.getSession();
      if (!session) {
        window.location.href = 'login.html';
        return;
      }
      // Data load will be wired in Task 4
      document.getElementById('loading-overlay').style.display = 'none';
      render();
    })();
```

Replace with:

```js
    (async function init() {
      const { data: { session } } = await sb.auth.getSession();
      if (!session) {
        window.location.href = 'login.html';
        return;
      }
      try {
        await loadFromSupabase();
      } catch (err) {
        console.error('Failed to load data:', err);
        document.getElementById('loading-overlay').innerHTML =
          'Failed to load data. Please refresh.';
        return;
      }
      document.getElementById('loading-overlay').style.display = 'none';
      render();
    })();
```

- [ ] **Step 3: Remove the now-unused STORAGE_KEY constant**

Find and delete this line (around line 367):

```js
    const STORAGE_KEY = 'swipeTracker';
```

- [ ] **Step 4: Remove the storage-warning HTML**

Find this block in the HTML (around line 60–63) and delete it:

```html
    <!-- ── Storage warning ─────────────────────── -->
    #storage-warning {
      background: #5a2000; border-bottom: 1px solid #8a4000;
      padding: 8px 20px; font-size: 12px; color: #ffb74d; display: none;
    }
```

Also find and delete the HTML element itself:

```html
  <div id="storage-warning">
    ...
  </div>
```

(Search for `id="storage-warning"` to find it in the HTML body.)

- [ ] **Step 5: Verify in browser**

Sign in via `login.html`. You should land on `index.html` with:
- "Loading…" overlay visible for a moment, then disappearing
- All 8 disciplines in the sidebar with `0/N` badges (fresh account, no entries)
- Global counts showing `0/47` and `0/18`
- No console errors

Open browser DevTools → Application → Local Storage — confirm the app no longer reads/writes localStorage.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: replace localStorage state layer with Supabase data load

loadFromSupabase() fetches entries, doctors, requirements in parallel.
First login seeds requirements from DEFAULT_REQUIREMENTS.
getState() is now a thin wrapper over module-level state variable."
```

---

## Task 5: Migrate saveEntry() to Supabase

**Files:**
- Modify: `index.html` (saveEntry function, ~lines 733–775)

- [ ] **Step 1: Replace saveEntry() with async Supabase version**

Find the existing `saveEntry()` function (starts around line 733):

```js
    function saveEntry() {
```

Replace the entire function with:

```js
    async function saveEntry() {
      if (!modalContext) return;

      const date = document.getElementById('f-date').value;
      if (!date) { alert('Please select a date.'); return; }

      let doctor = document.getElementById('f-doctor-select').value;
      if (doctor === '__new__') {
        const newName = document.getElementById('f-new-doctor').value.trim();
        if (!newName) { alert('Please enter a doctor name.'); return; }
        doctor = newName;
        if (!state.doctors.includes(doctor)) {
          const { error: docErr } = await sb.from('doctors').insert({ name: doctor });
          if (docErr) { alert('Failed to save doctor.'); return; }
          state.doctors.push(doctor);
        }
      }
      if (!doctor) { alert('Please select or add a doctor.'); return; }

      const { data, error } = await sb.from('entries').insert({
        discipline: modalContext.discipline,
        procedure: modalContext.proc,
        type: modalContext.type,
        date,
        doctor,
        axium_number: document.getElementById('f-axium').value.trim(),
        axium_form_code: document.getElementById('f-formcode').value.trim(),
        notes: document.getElementById('f-notes').value.trim(),
      }).select().single();

      if (error) { alert('Failed to save entry.'); return; }

      const entry = {
        id: data.id,
        discipline: data.discipline,
        procedure: data.procedure,
        type: data.type,
        date: data.date,
        doctor: data.doctor,
        axiUmNumber: data.axium_number || '',
        axiUmFormCode: data.axium_form_code || '',
        notes: data.notes || '',
        createdAt: new Date(data.created_at).getTime(),
      };

      state.entries.push(entry);
      closeModal();
      render();

      setTimeout(() => {
        const rid = rowId(entry.discipline, entry.procedure, entry.type);
        const row = document.getElementById(rid);
        if (row) toggleLogPanel(row);
      }, 50);
    }
```

- [ ] **Step 2: Verify in browser**

Sign in, navigate to Diagnosis, expand "NPI" formative, click "+ Log NPI formative". Fill in the modal:
- Date: today
- Doctor: click "Add new doctor…", type "Dr. Smith", press Save Entry

Verify:
- Entry appears in the expanded log panel immediately
- Counts update in sidebar and nav
- Open Supabase dashboard → Table Editor → `entries` — confirm the row exists with the correct `user_id`
- Open `doctors` table — confirm "Dr. Smith" is there

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: migrate saveEntry() to Supabase insert"
```

---

## Task 6: Migrate deleteEntry() to Supabase

**Files:**
- Modify: `index.html` (deleteEntry function, ~lines 616–632)

- [ ] **Step 1: Replace deleteEntry() with async Supabase version**

Find the existing `deleteEntry()` function:

```js
    function deleteEntry(id) {
      if (!confirm('Delete this entry?')) return;
      // Capture which row is open before render wipes the DOM
      const openRow = document.querySelector('.req-row.expanded');
      const ctx = openRow
        ? { disc: openRow.dataset.disc, proc: openRow.dataset.proc, type: openRow.dataset.type }
        : null;
      const state = getState();
      state.entries = state.entries.filter(e => e.id !== id);
      setState(state);
      render();
      // Reopen the same panel so the user doesn't lose their place
      if (ctx) {
        const row = document.getElementById(rowId(ctx.disc, ctx.proc, ctx.type));
        if (row) toggleLogPanel(row);
      }
    }
```

Replace with:

```js
    async function deleteEntry(id) {
      if (!confirm('Delete this entry?')) return;

      const openRow = document.querySelector('.req-row.expanded');
      const ctx = openRow
        ? { disc: openRow.dataset.disc, proc: openRow.dataset.proc, type: openRow.dataset.type }
        : null;

      const { error } = await sb.from('entries').delete().eq('id', id);
      if (error) { alert('Failed to delete entry.'); return; }

      state.entries = state.entries.filter(e => e.id !== id);
      render();

      if (ctx) {
        const row = document.getElementById(rowId(ctx.disc, ctx.proc, ctx.type));
        if (row) toggleLogPanel(row);
      }
    }
```

- [ ] **Step 2: Verify in browser**

Sign in, log an entry, expand the row, click "✕" on the entry, confirm deletion dialog, verify:
- Entry disappears from the log panel
- Counts update
- Supabase dashboard → `entries` table — row is gone

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: migrate deleteEntry() to Supabase delete"
```

---

## Task 7: Migrate doctors settings mutations to Supabase

**Files:**
- Modify: `index.html` (renderDoctorsSettings function ~lines 831–853; add-doctor-btn event listener ~lines 926–937)

- [ ] **Step 1: Replace renderDoctorsSettings() delete handler with async version**

Find the `renderDoctorsSettings()` function:

```js
    function renderDoctorsSettings() {
      const state = getState();
      const list = document.getElementById('doctors-list');
      if (state.doctors.length === 0) {
        list.innerHTML = '<div style="font-size:13px;color:var(--faint);font-style:italic;">No doctors saved yet.</div>';
      } else {
        list.innerHTML = state.doctors.map((d, i) => `
          <div class="doctor-item">
            <span>${esc(d)}</span>
            <button class="doctor-delete" data-index="${i}">✕ remove</button>
          </div>`).join('');

        list.querySelectorAll('.doctor-delete').forEach(btn => {
          btn.addEventListener('click', () => {
            const state = getState();
            const idx = parseInt(btn.dataset.index, 10);
            state.doctors = state.doctors.filter((_, i) => i !== idx);
            setState(state);
            renderDoctorsSettings();
          });
        });
      }
    }
```

Replace with:

```js
    function renderDoctorsSettings() {
      const list = document.getElementById('doctors-list');
      if (state.doctors.length === 0) {
        list.innerHTML = '<div style="font-size:13px;color:var(--faint);font-style:italic;">No doctors saved yet.</div>';
      } else {
        list.innerHTML = state.doctors.map((d, i) => `
          <div class="doctor-item">
            <span>${esc(d)}</span>
            <button class="doctor-delete" data-name="${esc(d)}">✕ remove</button>
          </div>`).join('');

        list.querySelectorAll('.doctor-delete').forEach(btn => {
          btn.addEventListener('click', async () => {
            const name = btn.dataset.name;
            const { error } = await sb.from('doctors').delete().eq('name', name);
            if (error) { alert('Failed to remove doctor.'); return; }
            state.doctors = state.doctors.filter(d => d !== name);
            renderDoctorsSettings();
          });
        });
      }
    }
```

Note: Using `data-name` instead of `data-index` so the delete is matched by name, not array position (safer).

- [ ] **Step 2: Replace the add-doctor-btn event listener with async version**

Find this block in the event listeners section (around line 926):

```js
    document.getElementById('add-doctor-btn').addEventListener('click', () => {
      const input = document.getElementById('new-doctor-input');
      const name = input.value.trim();
      if (!name) return;
      const state = getState();
      if (!state.doctors.includes(name)) {
        state.doctors.push(name);
        setState(state);
      }
      input.value = '';
      renderDoctorsSettings();
    });
```

Replace with:

```js
    document.getElementById('add-doctor-btn').addEventListener('click', async () => {
      const input = document.getElementById('new-doctor-input');
      const name = input.value.trim();
      if (!name) return;
      if (state.doctors.includes(name)) {
        input.value = '';
        return;
      }
      const { error } = await sb.from('doctors').insert({ name });
      if (error) { alert('Failed to add doctor.'); return; }
      state.doctors.push(name);
      input.value = '';
      renderDoctorsSettings();
    });
```

- [ ] **Step 3: Verify in browser**

Go to Settings → Doctors / Instructors. Add a doctor ("Dr. Jones"). Verify it appears. Remove it. Verify it disappears. Check Supabase `doctors` table to confirm add/delete both propagate.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: migrate doctors add/delete to Supabase"
```

---

## Task 8: Migrate requirements settings mutations to Supabase

**Files:**
- Modify: `index.html` (renderReqSettings function, ~lines 785–829)

- [ ] **Step 1: Replace the requirements input change handler with async version**

Find the `change` event listener inside `renderReqSettings()`:

```js
      tbody.querySelectorAll('.req-num-input').forEach(input => {
        input.addEventListener('change', () => {
          const state = getState();
          const disc = input.dataset.disc;
          const proc = input.dataset.proc;
          const field = input.dataset.field;
          const val = Math.max(0, parseInt(input.value, 10) || 0);
          input.value = val;
          if (state.requirements[disc] && state.requirements[disc][proc]) {
            state.requirements[disc][proc][field] = val;
            setState(state);
            renderNavCounts();
            renderSidebar();
          }
        });
      });
```

Replace with:

```js
      tbody.querySelectorAll('.req-num-input').forEach(input => {
        input.addEventListener('change', async () => {
          const disc = input.dataset.disc;
          const proc = input.dataset.proc;
          const field = input.dataset.field;
          const val = Math.max(0, parseInt(input.value, 10) || 0);
          input.value = val;

          if (!state.requirements[disc] || !state.requirements[disc][proc]) return;

          const { error } = await sb.from('requirements')
            .update({ [field]: val })
            .eq('discipline', disc)
            .eq('procedure', proc);

          if (error) {
            alert('Failed to save requirement change.');
            // Revert input to current state value
            input.value = state.requirements[disc][proc][field];
            return;
          }

          state.requirements[disc][proc][field] = val;
          renderNavCounts();
          renderSidebar();
        });
      });
```

- [ ] **Step 2: Verify in browser**

Go to Settings → Requirements. Change a formative count (e.g., Diagnosis → NPI from 2 to 3). Tab away. Verify:
- Nav bar counts update
- Sidebar badge updates
- Refresh the page — the new value (3) persists after reload from Supabase
- Supabase `requirements` table shows the updated row

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: migrate requirements edits to Supabase update"
```

---

## Task 9: End-to-end verification and GitHub Pages setup

**Files:**
- No code changes

- [ ] **Step 1: Full app walkthrough**

Sign in with a fresh account. Run through the full workflow:
1. Add entries across multiple disciplines (at least 3)
2. Delete one entry
3. Add a doctor in Settings, delete it
4. Change a requirement count in Settings, refresh page, confirm it persists
5. Log out — confirm redirect to `login.html`
6. Log back in — confirm all data is still there (entries, counts correct)
7. Create a second account in a separate incognito window — confirm it sees zero entries (data isolation)

- [ ] **Step 2: Push to GitHub**

Create a GitHub repository (e.g., `swipe-tracker`). Then:

```bash
git remote add origin https://github.com/YOUR_USERNAME/swipe-tracker.git
git branch -M main
git push -u origin main
```

- [ ] **Step 3: Enable GitHub Pages**

In the GitHub repo: Settings → Pages → Source: "Deploy from a branch" → Branch: `main`, Folder: `/ (root)` → Save.

Wait ~60 seconds for deployment. The app will be live at:
`https://YOUR_USERNAME.github.io/swipe-tracker/login.html`

- [ ] **Step 4: Test the live URL**

Open the GitHub Pages URL. Sign in, add an entry, verify it persists. Share the URL with another device and sign in — confirm the same data appears.

- [ ] **Step 5: Disable email confirmation (if not done in Task 1)**

In the Supabase dashboard: Authentication → Configuration → "Enable email confirmations" → toggle OFF. This ensures new users can sign in immediately after creating an account without needing to click a confirmation email.
