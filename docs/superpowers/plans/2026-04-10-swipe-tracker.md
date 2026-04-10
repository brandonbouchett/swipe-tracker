# Swipe Tracker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` file that lets a dental student log and track clinical graduation requirements across 8 disciplines, with data stored in `localStorage`.

**Architecture:** One `index.html` file — all HTML structure, CSS, and JavaScript inline. No build step, no server, no external dependencies. State lives in `localStorage` under the key `swipeTracker`. JavaScript is organized as plain functions in one `<script>` block: constants → state layer → render functions → event handlers → init call.

**Tech Stack:** Vanilla HTML5, CSS3, JavaScript (ES2020). `crypto.randomUUID()` for IDs. No frameworks, no libraries.

---

## File Map

| File | Purpose |
|---|---|
| `index.html` | The entire app — HTML, CSS, JS all inline |

---

### Task 1: HTML skeleton + CSS

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with full structure and styles**

Write the complete file. This is the scaffold every later task builds on.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Swipe Tracker</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --bg:        #0f0f1a;
      --surface:   #141428;
      --surface2:  #1a1a2e;
      --surface3:  #252550;
      --border:    #2a2a4a;
      --accent:    #4a90d9;
      --accent-lt: #6a9fd8;
      --text:      #e0e0e0;
      --muted:     #888;
      --faint:     #555;
      --green:     #4caf50;
      --orange:    #ff9800;
      --red:       #f44336;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: var(--bg);
      color: var(--text);
      height: 100vh;
      display: flex;
      flex-direction: column;
      overflow: hidden;
    }

    /* ── Top nav ─────────────────────────────── */
    #topnav {
      background: var(--surface2);
      border-bottom: 1px solid var(--border);
      padding: 0 20px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      height: 48px;
      flex-shrink: 0;
      gap: 16px;
    }
    #topnav .logo { font-weight: 700; font-size: 15px; color: var(--accent-lt); letter-spacing: 0.5px; }
    #topnav .nav-links { display: flex; gap: 4px; }
    #topnav .nav-link {
      padding: 6px 14px; border-radius: 6px; font-size: 13px;
      color: var(--muted); cursor: pointer; border: none; background: none;
      transition: background 0.15s, color 0.15s;
    }
    #topnav .nav-link:hover { background: var(--surface3); color: var(--text); }
    #topnav .nav-link.active { background: var(--surface3); color: var(--accent-lt); }
    #global-counts { font-size: 12px; color: var(--muted); white-space: nowrap; }

    /* ── Storage warning ─────────────────────── */
    #storage-warning {
      background: #5a2000; border-bottom: 1px solid #8a4000;
      padding: 8px 20px; font-size: 12px; color: #ffb74d; display: none;
    }

    /* ── App layout ──────────────────────────── */
    #app { display: flex; flex: 1; overflow: hidden; }

    /* ── Sidebar ─────────────────────────────── */
    #sidebar {
      width: 168px; background: var(--surface); border-right: 1px solid var(--border);
      overflow-y: auto; flex-shrink: 0; padding: 12px 8px;
    }
    .sidebar-label {
      font-size: 10px; color: var(--faint); text-transform: uppercase;
      letter-spacing: 1px; padding: 4px 8px; margin-bottom: 4px;
    }
    .disc-item {
      padding: 8px 10px; border-radius: 6px; font-size: 13px; cursor: pointer;
      display: flex; justify-content: space-between; align-items: center;
      margin-bottom: 2px; transition: background 0.12s;
    }
    .disc-item:hover { background: var(--surface2); }
    .disc-item.active { background: #1e2a4a; color: var(--accent-lt); }
    .disc-badge {
      font-size: 10px; padding: 1px 6px; border-radius: 8px;
      background: var(--surface3); color: var(--muted);
    }
    .disc-item.active .disc-badge { background: #1a3a6a; color: var(--accent-lt); }

    /* ── Main panel ──────────────────────────── */
    #main { flex: 1; overflow-y: auto; padding: 20px; }

    /* ── Views ───────────────────────────────── */
    .view { display: none; }
    .view.active { display: block; }

    /* ── Discipline header ───────────────────── */
    #disc-title { font-size: 20px; font-weight: 600; margin-bottom: 8px; }
    .legend { display: flex; gap: 16px; font-size: 11px; color: var(--muted); margin-bottom: 10px; }
    .legend span { display: flex; align-items: center; gap: 5px; }
    .dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; flex-shrink: 0; }
    .progress-wrap { background: var(--surface2); border-radius: 6px; height: 7px; overflow: hidden; margin-bottom: 6px; }
    .progress-bar { height: 100%; border-radius: 6px; background: linear-gradient(90deg, var(--accent), var(--accent-lt)); transition: width 0.3s; }
    #disc-progress-label { font-size: 12px; color: var(--muted); margin-bottom: 20px; }

    /* ── Section ─────────────────────────────── */
    .section { margin-bottom: 24px; }
    .section-title {
      font-size: 11px; text-transform: uppercase; letter-spacing: 1px;
      color: var(--accent-lt); margin-bottom: 10px; font-weight: 600;
    }

    /* ── Requirement table ───────────────────── */
    .req-table { background: var(--surface); border-radius: 8px; border: 1px solid var(--border); overflow: hidden; }
    .req-header {
      display: grid; grid-template-columns: 1fr 72px 72px 32px;
      padding: 7px 14px; gap: 8px; border-bottom: 1px solid var(--border);
    }
    .req-header span { font-size: 10px; color: var(--faint); text-transform: uppercase; letter-spacing: 0.5px; text-align: center; }
    .req-header span:first-child { text-align: left; }
    .req-row {
      display: grid; grid-template-columns: 1fr 72px 72px 32px;
      align-items: center; padding: 10px 14px; border-bottom: 1px solid var(--surface2);
      gap: 8px; cursor: pointer; transition: background 0.12s; user-select: none;
    }
    .req-row:last-of-type { border-bottom: none; }
    .req-row:hover, .req-row.expanded { background: #1a1a38; }
    .req-name { font-size: 13px; }
    .req-done { text-align: center; font-size: 13px; font-weight: 600; }
    .req-done.status-met    { color: var(--green); }
    .req-done.status-partial { color: var(--orange); }
    .req-done.status-none   { color: var(--red); }
    .req-needed { text-align: center; font-size: 13px; color: var(--muted); }
    .req-chevron { text-align: center; font-size: 11px; color: var(--faint); }

    /* ── Log panel ───────────────────────────── */
    .log-panel {
      background: #0f1020; border-bottom: 1px solid var(--border);
      padding: 12px 14px; display: none;
    }
    .log-panel.open { display: block; }
    .log-entry {
      display: flex; justify-content: space-between; align-items: center;
      padding: 7px 10px; background: var(--surface); border-radius: 5px;
      margin-bottom: 5px; font-size: 12px; gap: 8px;
    }
    .entry-info { display: flex; gap: 10px; align-items: center; flex-wrap: wrap; color: var(--muted); }
    .entry-date { color: var(--accent-lt); font-weight: 600; }
    .entry-doctor { color: var(--text); }
    .entry-axium { color: var(--faint); }
    .entry-notes { color: var(--faint); font-style: italic; max-width: 200px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    .entry-delete {
      color: var(--faint); cursor: pointer; font-size: 11px;
      padding: 2px 6px; border-radius: 3px; border: none; background: none;
      flex-shrink: 0; transition: color 0.12s;
    }
    .entry-delete:hover { color: var(--red); }
    .log-empty { font-size: 12px; color: var(--faint); font-style: italic; margin-bottom: 8px; }
    .add-entry-btn {
      margin-top: 8px; padding: 7px 14px; background: var(--surface2);
      border: 1px dashed #2a4a7a; border-radius: 6px; color: var(--accent-lt);
      font-size: 12px; cursor: pointer; width: 100%; text-align: center;
      transition: background 0.12s;
    }
    .add-entry-btn:hover { background: var(--surface3); }

    /* ── Modal ───────────────────────────────── */
    #modal-overlay {
      position: fixed; inset: 0; background: rgba(0,0,0,0.65);
      display: none; align-items: center; justify-content: center; z-index: 100;
    }
    #modal-overlay.open { display: flex; }
    #modal {
      background: var(--surface2); border: 1px solid var(--border);
      border-radius: 10px; padding: 24px; width: 420px; max-width: 95vw;
    }
    #modal h3 { font-size: 15px; font-weight: 600; margin-bottom: 18px; }
    .form-row { margin-bottom: 14px; }
    .form-row label { display: block; font-size: 11px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 5px; }
    .form-row input,
    .form-row select,
    .form-row textarea {
      width: 100%; background: var(--bg); border: 1px solid var(--border);
      border-radius: 6px; padding: 8px 10px; color: var(--text);
      font-size: 13px; font-family: inherit; outline: none;
      transition: border-color 0.15s;
    }
    .form-row input:focus,
    .form-row select:focus,
    .form-row textarea:focus { border-color: var(--accent); }
    .form-row textarea { height: 70px; resize: none; }
    .form-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
    .modal-actions { display: flex; justify-content: flex-end; gap: 8px; margin-top: 18px; }
    .btn { padding: 8px 18px; border-radius: 6px; font-size: 13px; cursor: pointer; border: none; font-family: inherit; }
    .btn-primary { background: #2a5aaa; color: white; }
    .btn-primary:hover { background: #3468c0; }
    .btn-secondary { background: var(--surface3); color: var(--muted); }
    .btn-secondary:hover { color: var(--text); }

    /* ── Settings page ───────────────────────── */
    #settings-view h2 { font-size: 18px; font-weight: 600; margin-bottom: 6px; }
    #settings-view .settings-subtitle { font-size: 13px; color: var(--muted); margin-bottom: 24px; }
    .settings-section { margin-bottom: 32px; }
    .settings-section h3 { font-size: 13px; text-transform: uppercase; letter-spacing: 1px; color: var(--accent-lt); margin-bottom: 12px; }
    .settings-table { width: 100%; border-collapse: collapse; font-size: 13px; }
    .settings-table th { text-align: left; padding: 6px 10px; font-size: 10px; text-transform: uppercase; letter-spacing: 0.5px; color: var(--faint); border-bottom: 1px solid var(--border); }
    .settings-table td { padding: 8px 10px; border-bottom: 1px solid var(--surface2); vertical-align: middle; }
    .settings-table tr:last-child td { border-bottom: none; }
    .settings-table .disc-col { color: var(--muted); font-size: 11px; }
    .req-num-input {
      width: 56px; background: var(--bg); border: 1px solid var(--border);
      border-radius: 5px; padding: 4px 8px; color: var(--text);
      font-size: 13px; text-align: center; font-family: inherit;
    }
    .req-num-input:focus { outline: none; border-color: var(--accent); }
    .doctors-list { display: flex; flex-direction: column; gap: 6px; max-width: 400px; }
    .doctor-item {
      display: flex; justify-content: space-between; align-items: center;
      background: var(--surface); padding: 8px 12px; border-radius: 6px; font-size: 13px;
    }
    .doctor-delete { color: var(--faint); cursor: pointer; font-size: 11px; border: none; background: none; }
    .doctor-delete:hover { color: var(--red); }
    .add-doctor-row { display: flex; gap: 8px; margin-top: 8px; max-width: 400px; }
    .add-doctor-row input { flex: 1; background: var(--bg); border: 1px solid var(--border); border-radius: 6px; padding: 8px 10px; color: var(--text); font-size: 13px; font-family: inherit; outline: none; }
    .add-doctor-row input:focus { border-color: var(--accent); }
  </style>
</head>
<body>

  <!-- Top navigation -->
  <nav id="topnav">
    <div class="logo">Swipe Tracker</div>
    <div class="nav-links">
      <button class="nav-link active" data-view="tracker">Tracker</button>
      <button class="nav-link" data-view="settings">Settings</button>
    </div>
    <div id="global-counts">Loading…</div>
  </nav>

  <!-- Storage warning banner -->
  <div id="storage-warning">
    ⚠️ localStorage is unavailable — data will not be saved. Try opening this file outside of private/incognito mode.
  </div>

  <div id="app">
    <!-- Sidebar -->
    <nav id="sidebar">
      <div class="sidebar-label">Disciplines</div>
      <div id="disc-list"><!-- rendered by JS --></div>
    </nav>

    <!-- Main panel -->
    <main id="main">

      <!-- Tracker view -->
      <div id="tracker-view" class="view active">
        <div id="disc-title"></div>
        <div class="legend">
          <span><span class="dot" style="background:var(--green)"></span> Complete</span>
          <span><span class="dot" style="background:var(--orange)"></span> In progress</span>
          <span><span class="dot" style="background:var(--red)"></span> Not started</span>
        </div>
        <div class="progress-wrap"><div class="progress-bar" id="disc-progress-bar"></div></div>
        <div id="disc-progress-label"></div>

        <div id="formatives-section" class="section">
          <div class="section-title">Formatives</div>
          <div class="req-table" id="formatives-table">
            <div class="req-header">
              <span>Procedure</span><span>Done</span><span>Needed</span><span></span>
            </div>
            <div id="formatives-rows"><!-- rendered by JS --></div>
          </div>
        </div>

        <div id="summatives-section" class="section">
          <div class="section-title">Summatives</div>
          <div class="req-table" id="summatives-table">
            <div class="req-header">
              <span>Procedure</span><span>Done</span><span>Needed</span><span></span>
            </div>
            <div id="summatives-rows"><!-- rendered by JS --></div>
          </div>
        </div>
      </div>

      <!-- Settings view -->
      <div id="settings-view" class="view">
        <h2>Settings</h2>
        <p class="settings-subtitle">Edit required counts or manage your saved doctors list.</p>

        <div class="settings-section">
          <h3>Required Counts</h3>
          <table class="settings-table" id="req-settings-table">
            <thead>
              <tr>
                <th>Discipline</th>
                <th>Procedure</th>
                <th style="text-align:center">Form.</th>
                <th style="text-align:center">Summ.</th>
              </tr>
            </thead>
            <tbody id="req-settings-body"><!-- rendered by JS --></tbody>
          </table>
        </div>

        <div class="settings-section">
          <h3>Doctors / Instructors</h3>
          <div class="doctors-list" id="doctors-list"><!-- rendered by JS --></div>
          <div class="add-doctor-row">
            <input type="text" id="new-doctor-input" placeholder="Dr. Name…">
            <button class="btn btn-primary" id="add-doctor-btn">Add</button>
          </div>
        </div>
      </div>

    </main>
  </div>

  <!-- Add Entry Modal -->
  <div id="modal-overlay">
    <div id="modal">
      <h3 id="modal-title">Log Entry</h3>
      <div class="form-grid">
        <div class="form-row">
          <label>Date</label>
          <input type="date" id="f-date">
        </div>
        <div class="form-row">
          <label>axiUm #</label>
          <input type="text" id="f-axium" placeholder="e.g. 50100511">
        </div>
      </div>
      <div class="form-row">
        <label>Doctor / Instructor</label>
        <select id="f-doctor-select"></select>
      </div>
      <div class="form-row" id="new-doctor-row" style="display:none">
        <label>New doctor name</label>
        <input type="text" id="f-new-doctor" placeholder="Dr. …">
      </div>
      <div class="form-row">
        <label>axiUm Form Code</label>
        <input type="text" id="f-formcode" placeholder="e.g. D0150">
      </div>
      <div class="form-row">
        <label>Notes</label>
        <textarea id="f-notes" placeholder="Optional notes…"></textarea>
      </div>
      <div class="modal-actions">
        <button class="btn btn-secondary" id="modal-cancel">Cancel</button>
        <button class="btn btn-primary" id="modal-save">Save Entry</button>
      </div>
    </div>
  </div>

  <script>
    // ── JS goes here in later tasks ──
  </script>
</body>
</html>
```

- [ ] **Step 2: Open `index.html` in a browser**

Double-click the file. You should see:
- Dark background page with "Swipe Tracker" in the top nav
- "Tracker" and "Settings" nav buttons (no functionality yet)
- Empty sidebar and main panel
- No errors in the browser console (open with F12)

---

### Task 2: Data layer — constants, state, seed requirements

**Files:**
- Modify: `index.html` — replace `// ── JS goes here in later tasks ──` with the script below

- [ ] **Step 1: Add constants and state functions**

Replace the `<script>` block content with:

```javascript
// ─────────────────────────────────────────────
// CONSTANTS
// ─────────────────────────────────────────────
const STORAGE_KEY = 'swipeTracker';
const DISCIPLINES = ['Diagnosis','Operative','Peds','Perio','Endo','Prosth','Surgery','Ortho'];

const DEFAULT_REQUIREMENTS = {
  Diagnosis: {
    'Tx Plan Basic':                          { formative: 1, summative: 0 },
    'Tx Plan Complex':                        { formative: 4, summative: 0 },
    'NPI':                                    { formative: 2, summative: 1 },
    'Completion of Comp Care':                { formative: 2, summative: 0 },
    'Emergency':                              { formative: 2, summative: 1 },
    'Special Care':                           { formative: 4, summative: 1 },
    'Collab w Dental Hygienist':              { formative: 2, summative: 0 },
    'Patient Assessment':                     { formative: 0, summative: 1 },
    'Outcome of Comp Tx & Recall':            { formative: 0, summative: 1 },
    'Interprofessional Comm & Pt Presentation': { formative: 0, summative: 1 },
    'Caries Management':                      { formative: 5, summative: 1 },
  },
  Operative: {
    'Total Fillings':    { formative: 5, summative: 0 },
    'Class I':           { formative: 1, summative: 1 },
    'Class II':          { formative: 1, summative: 1 },
    'Class III/IV':      { formative: 1, summative: 1 },
    'Class V':           { formative: 1, summative: 1 },
    'Special Care':      { formative: 4, summative: 1 },
  },
  Peds: {
    'New/Recall Exam':   { formative: 4, summative: 1 },
    'Prophy':            { formative: 4, summative: 0 },
    'Fluoride':          { formative: 4, summative: 0 },
    'Bitewings (x2)':   { formative: 3, summative: 0 },
    'Xray on Peds Patient': { formative: 1, summative: 0 },
    'Class I/II':        { formative: 2, summative: 1 },
    'Class III/IV/V':    { formative: 2, summative: 0 },
    'Sealant':           { formative: 4, summative: 1 },
    'Infant Oral Exam':  { formative: 2, summative: 0 },
    'Peds Case Completion': { formative: 2, summative: 0 },
    'Complex':           { formative: 1, summative: 0 },
    'Pulp Therapy':      { formative: 1, summative: 0 },
  },
  Perio: {
    'Assists':           { formative: 4, summative: 0 },
    'Perio Chart':       { formative: 6, summative: 2 },
    'Perio Diagnosis':   { formative: 7, summative: 1 },
    'Prophy':            { formative: 7, summative: 1 },
    'SRP (Full)':        { formative: 9, summative: 1 },
    'Perio Surgery':     { formative: 3, summative: 1 },
    'Perio Maintenance': { formative: 5, summative: 1 },
    'Perio Re-Eval':     { formative: 5, summative: 1 },
  },
  Endo: {
    'Assists':           { formative: 4, summative: 0 },
    'Case Difficulty':   { formative: 5, summative: 0 },
    'Endo Diagnosis':    { formative: 5, summative: 0 },
    'NonSurg Endo':      { formative: 4, summative: 1 },
    'Endo Recall':       { formative: 2, summative: 1 },
  },
  Prosth: {
    'Single Crown':      { formative: 6, summative: 1 },
    'Fixed Partial':     { formative: 1, summative: 1 },
    'Complete Denture':  { formative: 1, summative: 1 },
    'Removable Partial': { formative: 1, summative: 1 },
    'Lab Prescription':  { formative: 6, summative: 1 },
    'Implant Crown':     { formative: 1, summative: 1 },
  },
  Surgery: {
    'Presurg Case':                { formative: 2, summative: 0 },
    'Simple/Surgical Extraction':  { formative: 5, summative: 1 },
    'Multiple Tooth Extraction':   { formative: 1, summative: 1 },
    'Inhalation Sedation':         { formative: 2, summative: 1 },
    'Prescription Writing':        { formative: 0, summative: 1 },
    'Medical Risk Assessment':     { formative: 0, summative: 1 },
    'Local Anesthesia':            { formative: 0, summative: 1 },
  },
  Ortho: {
    'Consultation':      { formative: 3, summative: 1 },
    'TMD':               { formative: 0, summative: 1 },
  },
};

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
  // Deep-clone DEFAULT_REQUIREMENTS so edits don't mutate the constant
  return {
    requirements: JSON.parse(JSON.stringify(DEFAULT_REQUIREMENTS)),
    entries: [],
    doctors: [],
  };
}

function generateId() {
  return crypto.randomUUID();
}

// ─────────────────────────────────────────────
// QUERY HELPERS
// ─────────────────────────────────────────────
function getEntriesFor(state, discipline, procedure, type) {
  return state.entries.filter(
    e => e.discipline === discipline && e.procedure === procedure && e.type === type
  );
}

function getCountFor(state, discipline, procedure, type) {
  return getEntriesFor(state, discipline, procedure, type).length;
}

function computeGlobalProgress(state) {
  let formDone = 0, formNeeded = 0, summDone = 0, summNeeded = 0;
  for (const [disc, procs] of Object.entries(state.requirements)) {
    for (const [proc, req] of Object.entries(procs)) {
      formNeeded += req.formative;
      summNeeded += req.summative;
      formDone += Math.min(getCountFor(state, disc, proc, 'formative'), req.formative);
      summDone += Math.min(getCountFor(state, disc, proc, 'summative'), req.summative);
    }
  }
  return { formDone, formNeeded, summDone, summNeeded };
}

function computeDiscProgress(state, discipline) {
  const procs = state.requirements[discipline] || {};
  let done = 0, needed = 0;
  for (const [proc, req] of Object.entries(procs)) {
    needed += req.formative + req.summative;
    done += Math.min(getCountFor(state, discipline, proc, 'formative'), req.formative);
    done += Math.min(getCountFor(state, discipline, proc, 'summative'), req.summative);
  }
  return { done, needed };
}
```

- [ ] **Step 2: Verify in browser**

Refresh `index.html`. Open the browser console (F12 → Console). Type:

```javascript
getState()
```

Expected: an object with `requirements`, `entries: []`, and `doctors: []`. The `requirements` object should have all 8 discipline keys with their procedure lists.

---

### Task 3: Sidebar + nav rendering

**Files:**
- Modify: `index.html` — append to the `<script>` block

- [ ] **Step 1: Append rendering functions for the nav and sidebar**

Add after the query helpers:

```javascript
// ─────────────────────────────────────────────
// RENDERING — NAV + SIDEBAR
// ─────────────────────────────────────────────
let activeDiscipline = DISCIPLINES[0];

function renderNavCounts() {
  const state = getState();
  const { formDone, formNeeded, summDone, summNeeded } = computeGlobalProgress(state);
  document.getElementById('global-counts').textContent =
    `Formatives: ${formDone}/${formNeeded}  |  Summatives: ${summDone}/${summNeeded}`;
}

function renderSidebar() {
  const state = getState();
  const container = document.getElementById('disc-list');
  container.innerHTML = DISCIPLINES.map(disc => {
    const { done, needed } = computeDiscProgress(state, disc);
    const isActive = disc === activeDiscipline;
    return `
      <div class="disc-item${isActive ? ' active' : ''}" data-disc="${disc}">
        <span>${disc}</span>
        <span class="disc-badge">${done}/${needed}</span>
      </div>`;
  }).join('');

  container.querySelectorAll('.disc-item').forEach(el => {
    el.addEventListener('click', () => {
      activeDiscipline = el.dataset.disc;
      render();
    });
  });
}
```

- [ ] **Step 2: Verify in browser**

Refresh. Open console and run:

```javascript
renderSidebar(); renderNavCounts();
```

Expected: sidebar shows all 8 disciplines with `0/N` badges, nav shows `Formatives: 0/… | Summatives: 0/…`.

---

### Task 4: Discipline detail view

**Files:**
- Modify: `index.html` — append to the `<script>` block

- [ ] **Step 1: Append the discipline rendering function**

```javascript
// ─────────────────────────────────────────────
// RENDERING — DISCIPLINE DETAIL
// ─────────────────────────────────────────────
function statusClass(done, needed) {
  if (needed === 0) return '';
  if (done >= needed) return 'status-met';
  if (done > 0) return 'status-partial';
  return 'status-none';
}

function buildReqRows(state, discipline, type) {
  const procs = state.requirements[discipline] || {};
  const rows = [];
  for (const [proc, req] of Object.entries(procs)) {
    const needed = req[type];
    if (needed === 0) continue;
    const done = getCountFor(state, discipline, proc, type);
    const sc = statusClass(done, needed);
    const rowId = `row-${discipline}-${proc}-${type}`.replace(/[^a-z0-9-]/gi, '_');
    const panelId = `panel-${discipline}-${proc}-${type}`.replace(/[^a-z0-9-]/gi, '_');
    rows.push(`
      <div class="req-row" id="${rowId}"
           data-disc="${discipline}" data-proc="${proc}" data-type="${type}">
        <div class="req-name">${proc}</div>
        <div class="req-done ${sc}">${done}</div>
        <div class="req-needed">${needed}</div>
        <div class="req-chevron">▶</div>
      </div>
      <div class="log-panel" id="${panelId}"></div>`);
  }
  return rows.join('');
}

function renderDiscipline() {
  const state = getState();
  const disc = activeDiscipline;
  document.getElementById('disc-title').textContent = disc;

  const { done, needed } = computeDiscProgress(state, disc);
  const pct = needed > 0 ? Math.round((done / needed) * 100) : 0;
  document.getElementById('disc-progress-bar').style.width = pct + '%';
  document.getElementById('disc-progress-label').textContent =
    `${done} of ${needed} requirements met`;

  document.getElementById('formatives-rows').innerHTML =
    buildReqRows(state, disc, 'formative');
  document.getElementById('summatives-rows').innerHTML =
    buildReqRows(state, disc, 'summative');

  // Hide section if it has no rows
  const fRows = document.getElementById('formatives-rows').children.length;
  document.getElementById('formatives-section').style.display = fRows ? '' : 'none';
  const sRows = document.getElementById('summatives-rows').children.length;
  document.getElementById('summatives-section').style.display = sRows ? '' : 'none';

  // Attach row click handlers
  document.querySelectorAll('.req-row').forEach(row => {
    row.addEventListener('click', () => toggleLogPanel(row));
  });
}
```

- [ ] **Step 2: Append master render + nav switching functions**

```javascript
// ─────────────────────────────────────────────
// MASTER RENDER + VIEW SWITCHING
// ─────────────────────────────────────────────
function render() {
  renderNavCounts();
  renderSidebar();
  renderDiscipline();
}

function showView(viewName) {
  document.querySelectorAll('.view').forEach(v => v.classList.remove('active'));
  document.getElementById(viewName + '-view').classList.add('active');
  document.querySelectorAll('.nav-link').forEach(btn => {
    btn.classList.toggle('active', btn.dataset.view === viewName);
  });
  if (viewName === 'settings') renderSettings();
}
```

- [ ] **Step 3: Wire nav buttons and init**

```javascript
// ─────────────────────────────────────────────
// INIT
// ─────────────────────────────────────────────
document.querySelectorAll('.nav-link').forEach(btn => {
  btn.addEventListener('click', () => showView(btn.dataset.view));
});

// Seed localStorage on first ever load
(function init() {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) setState(buildDefaultState());
  if (!storageAvailable) {
    document.getElementById('storage-warning').style.display = 'block';
  }
  render();
})();
```

- [ ] **Step 4: Verify in browser**

Refresh. Expected:
- Sidebar shows all 8 disciplines with `0/N` badges
- Main panel shows "Diagnosis" with Formatives and Summatives tables
- All `Done` counts show 0 in red
- Clicking a different discipline in the sidebar updates the main panel
- Clicking "Settings" nav link shows the settings view (empty for now)

---

### Task 5: Expandable rows + entry log + delete

**Files:**
- Modify: `index.html` — append to the `<script>` block

- [ ] **Step 1: Append log panel rendering and toggle**

```javascript
// ─────────────────────────────────────────────
// LOG PANEL
// ─────────────────────────────────────────────
function sanitizeId(str) {
  return str.replace(/[^a-z0-9-]/gi, '_');
}

function panelId(discipline, proc, type) {
  return `panel-${sanitizeId(discipline)}-${sanitizeId(proc)}-${sanitizeId(type)}`;
}

function rowId(discipline, proc, type) {
  return `row-${sanitizeId(discipline)}-${sanitizeId(proc)}-${sanitizeId(type)}`;
}

function renderLogPanel(discipline, proc, type) {
  const state = getState();
  const entries = getEntriesFor(state, discipline, proc, type);
  const pid = panelId(discipline, proc, type);
  const panel = document.getElementById(pid);
  if (!panel) return;

  const entriesHtml = entries.length === 0
    ? `<div class="log-empty">No entries yet.</div>`
    : entries.map(e => `
        <div class="log-entry">
          <div class="entry-info">
            <span class="entry-date">${e.date || '—'}</span>
            <span class="entry-doctor">${e.doctor || '—'}</span>
            ${e.axiUmNumber ? `<span class="entry-axium">axiUm #${e.axiUmNumber}</span>` : ''}
            ${e.axiUmFormCode ? `<span class="entry-axium">${e.axiUmFormCode}</span>` : ''}
            ${e.notes ? `<span class="entry-notes" title="${e.notes}">${e.notes}</span>` : ''}
          </div>
          <button class="entry-delete" data-id="${e.id}">✕</button>
        </div>`).join('');

  panel.innerHTML = `
    ${entriesHtml}
    <button class="add-entry-btn"
            data-disc="${discipline}" data-proc="${proc}" data-type="${type}">
      + Log ${proc} ${type}
    </button>`;

  panel.querySelector('.add-entry-btn').addEventListener('click', (ev) => {
    ev.stopPropagation();
    openModal(discipline, proc, type);
  });

  panel.querySelectorAll('.entry-delete').forEach(btn => {
    btn.addEventListener('click', (ev) => {
      ev.stopPropagation();
      deleteEntry(btn.dataset.id);
    });
  });
}

function toggleLogPanel(row) {
  const disc = row.dataset.disc;
  const proc = row.dataset.proc;
  const type = row.dataset.type;
  const pid = panelId(disc, proc, type);
  const panel = document.getElementById(pid);
  if (!panel) return;

  const isOpen = panel.classList.contains('open');
  // Close all open panels
  document.querySelectorAll('.log-panel.open').forEach(p => p.classList.remove('open'));
  document.querySelectorAll('.req-row.expanded').forEach(r => {
    r.classList.remove('expanded');
    r.querySelector('.req-chevron').textContent = '▶';
  });

  if (!isOpen) {
    panel.classList.add('open');
    row.classList.add('expanded');
    row.querySelector('.req-chevron').textContent = '▼';
    renderLogPanel(disc, proc, type);
  }
}

function deleteEntry(id) {
  if (!confirm('Delete this entry?')) return;
  const state = getState();
  state.entries = state.entries.filter(e => e.id !== id);
  setState(state);
  render();
  // Re-open the panel that was showing (find it by the entry that was just deleted)
  // render() closes all panels, which is fine — user can re-expand if needed
}
```

- [ ] **Step 2: Verify in browser**

Refresh. Click any procedure row. Expected:
- Row expands downward, showing "No entries yet." and a "+ Log [proc] [type]" button
- Chevron flips from ▶ to ▼
- Clicking the same row again collapses it
- Clicking a different row collapses the first and opens the new one

---

### Task 6: Add Entry modal

**Files:**
- Modify: `index.html` — append to the `<script>` block

- [ ] **Step 1: Append modal open/close and doctor dropdown logic**

```javascript
// ─────────────────────────────────────────────
// ADD ENTRY MODAL
// ─────────────────────────────────────────────
let modalContext = null; // { discipline, proc, type }

function openModal(discipline, proc, type) {
  modalContext = { discipline, proc, type };
  document.getElementById('modal-title').textContent =
    `Log ${proc} — ${type.charAt(0).toUpperCase() + type.slice(1)}`;

  // Default date to today
  document.getElementById('f-date').value = new Date().toISOString().slice(0, 10);
  document.getElementById('f-axium').value = '';
  document.getElementById('f-formcode').value = '';
  document.getElementById('f-notes').value = '';

  populateDoctorSelect();
  document.getElementById('new-doctor-row').style.display = 'none';
  document.getElementById('modal-overlay').classList.add('open');
}

function closeModal() {
  document.getElementById('modal-overlay').classList.remove('open');
  modalContext = null;
}

function populateDoctorSelect() {
  const state = getState();
  const select = document.getElementById('f-doctor-select');
  const doctors = state.doctors;
  select.innerHTML = [
    doctors.length === 0 ? '<option value="">— no saved doctors —</option>' : '',
    ...doctors.map(d => `<option value="${d}">${d}</option>`),
    '<option value="__new__">+ Add new doctor…</option>',
  ].join('');
  // Reset to first real doctor if available
  if (doctors.length > 0) select.value = doctors[0];
}

function saveEntry() {
  if (!modalContext) return;
  const state = getState();

  let doctor = document.getElementById('f-doctor-select').value;
  if (doctor === '__new__') {
    const newName = document.getElementById('f-new-doctor').value.trim();
    if (!newName) { alert('Please enter a doctor name.'); return; }
    doctor = newName;
    if (!state.doctors.includes(doctor)) {
      state.doctors.push(doctor);
    }
  }

  const entry = {
    id: generateId(),
    discipline: modalContext.discipline,
    procedure: modalContext.proc,
    type: modalContext.type,
    date: document.getElementById('f-date').value,
    doctor,
    axiUmNumber: document.getElementById('f-axium').value.trim(),
    axiUmFormCode: document.getElementById('f-formcode').value.trim(),
    notes: document.getElementById('f-notes').value.trim(),
    createdAt: Date.now(),
  };

  state.entries.push(entry);
  setState(state);
  closeModal();
  render();

  // Re-open the row that was being logged (for UX continuity)
  setTimeout(() => {
    const rid = rowId(entry.discipline, entry.procedure, entry.type);
    const row = document.getElementById(rid);
    if (row) toggleLogPanel(row);
  }, 50);
}
```

- [ ] **Step 2: Append modal event listeners**

```javascript
document.getElementById('modal-cancel').addEventListener('click', closeModal);
document.getElementById('modal-save').addEventListener('click', saveEntry);
document.getElementById('modal-overlay').addEventListener('click', e => {
  if (e.target === document.getElementById('modal-overlay')) closeModal();
});

document.getElementById('f-doctor-select').addEventListener('change', function () {
  const isNew = this.value === '__new__';
  document.getElementById('new-doctor-row').style.display = isNew ? '' : 'none';
  if (isNew) document.getElementById('f-new-doctor').focus();
});
```

- [ ] **Step 3: Verify in browser**

Refresh. Expand a procedure row and click the "+ Log" button. Expected:
- Modal opens with today's date pre-filled
- If no doctors saved yet, shows "— no saved doctors —" and "+ Add new doctor…"
- Selecting "+ Add new doctor…" reveals a text input
- Clicking Cancel closes the modal
- Clicking Save with a new doctor name: modal closes, entry appears in the log panel, counts update in the sidebar badge and nav bar

---

### Task 7: Settings page

**Files:**
- Modify: `index.html` — append to the `<script>` block

- [ ] **Step 1: Append settings rendering**

```javascript
// ─────────────────────────────────────────────
// SETTINGS
// ─────────────────────────────────────────────
function renderSettings() {
  renderReqSettings();
  renderDoctorsSettings();
}

function renderReqSettings() {
  const state = getState();
  const tbody = document.getElementById('req-settings-body');
  const rows = [];
  for (const disc of DISCIPLINES) {
    const procs = state.requirements[disc] || {};
    let first = true;
    for (const [proc, req] of Object.entries(procs)) {
      const safeDisc = sanitizeId(disc);
      const safeProc = sanitizeId(proc);
      rows.push(`
        <tr>
          <td class="disc-col">${first ? disc : ''}</td>
          <td>${proc}</td>
          <td style="text-align:center">
            <input class="req-num-input" type="number" min="0" max="99"
                   data-disc="${disc}" data-proc="${proc}" data-field="formative"
                   value="${req.formative}">
          </td>
          <td style="text-align:center">
            <input class="req-num-input" type="number" min="0" max="99"
                   data-disc="${disc}" data-proc="${proc}" data-field="summative"
                   value="${req.summative}">
          </td>
        </tr>`);
      first = false;
    }
  }
  tbody.innerHTML = rows.join('');

  tbody.querySelectorAll('.req-num-input').forEach(input => {
    input.addEventListener('change', () => {
      const state = getState();
      const disc = input.dataset.disc;
      const proc = input.dataset.proc;
      const field = input.dataset.field;
      const val = Math.max(0, parseInt(input.value, 10) || 0);
      input.value = val;
      state.requirements[disc][proc][field] = val;
      setState(state);
      renderNavCounts();
      renderSidebar();
    });
  });
}

function renderDoctorsSettings() {
  const state = getState();
  const list = document.getElementById('doctors-list');
  if (state.doctors.length === 0) {
    list.innerHTML = '<div style="font-size:13px;color:var(--faint);font-style:italic;">No doctors saved yet.</div>';
  } else {
    list.innerHTML = state.doctors.map((d, i) => `
      <div class="doctor-item">
        <span>${d}</span>
        <button class="doctor-delete" data-index="${i}">✕ remove</button>
      </div>`).join('');

    list.querySelectorAll('.doctor-delete').forEach(btn => {
      btn.addEventListener('click', () => {
        const state = getState();
        state.doctors.splice(parseInt(btn.dataset.index, 10), 1);
        setState(state);
        renderDoctorsSettings();
      });
    });
  }
}

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

document.getElementById('new-doctor-input').addEventListener('keydown', e => {
  if (e.key === 'Enter') document.getElementById('add-doctor-btn').click();
});
```

- [ ] **Step 2: Verify in browser**

Click "Settings" nav link. Expected:
- Table shows all 55+ procedures grouped by discipline with editable number inputs
- Changing a number and tabbing away updates it in localStorage (open console, run `getState().requirements.Diagnosis` to confirm)
- Doctors section shows "No doctors saved yet." if none added; otherwise lists them with remove buttons
- Adding a doctor via the text field + Add button or Enter key adds them to the list
- Newly added doctors appear in the modal dropdown next time it's opened

---

### Task 8: Final wiring and polish

**Files:**
- Modify: `index.html` — small targeted additions

- [ ] **Step 1: Check localStorage availability on load**

In the `init()` IIFE (already in the script), verify the storage-warning logic is in place — it should already be there from Task 2. Confirm by opening the file in a private/incognito window (Safari: File > New Private Window; Chrome: Cmd+Shift+N). Expected: yellow warning banner appears at the top.

- [ ] **Step 2: Verify full end-to-end flow**

Run through this checklist in the browser:

1. Open `index.html` fresh (clear localStorage first: console → `localStorage.clear()`, then refresh)
2. All counts show 0/N in red
3. Click "Diagnosis" in sidebar → main panel shows Diagnosis
4. Expand "NPI" formative row → shows "No entries yet."
5. Click "+ Log NPI formative" → modal opens with today's date
6. Select "+ Add new doctor…" → doctor name field appears
7. Type "Dr. Smith", fill axiUm # as `12345`, add a note, click Save
8. Entry appears in the log panel with correct fields
9. Sidebar badge for Diagnosis updates (e.g. 1/N)
10. Nav bar updates (Formatives: 1/…)
11. Log another NPI formative with same doctor → Done count shows 2 in green
12. Click Settings → "Dr. Smith" appears in doctors list
13. Change NPI formative requirement to 3 → sidebar badge decreases
14. Change it back to 2 → sidebar badge returns to "met"
15. Delete one entry → count goes back to 1, entry is gone from log
16. Reload page → all data persists

- [ ] **Step 3: Done**

The app is complete. Open `index.html` and start tracking.

---

## Self-Review

**Spec coverage:**
- [x] Single HTML file with localStorage ✓
- [x] 8 disciplines, all procedures with correct required counts ✓
- [x] Fields: date, doctor, axiUm #, axiUm form code, notes ✓
- [x] Layout C: sidebar + detail panel ✓
- [x] Doctor dropdown + "add new" inline ✓
- [x] Entry log per procedure with delete ✓
- [x] Settings page with editable requirements + doctors management ✓
- [x] Color coding (green/orange/red) ✓
- [x] Progress bar per discipline ✓
- [x] Global counts in nav bar ✓
- [x] localStorage unavailability warning ✓

**Placeholder scan:** No TBDs, no "similar to above", all code blocks complete.

**Type consistency:** `getState()` returns `{ requirements, entries, doctors }` — used consistently throughout. `entry` shape (`id, discipline, procedure, type, date, doctor, axiUmNumber, axiUmFormCode, notes, createdAt`) defined in Task 6 and read in Task 5 — keys match. `sanitizeId()` defined in Task 5, used in Tasks 5 and 7 — consistent.
