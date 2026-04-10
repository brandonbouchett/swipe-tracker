# Swipe Tracker — Design Spec
**Date:** 2026-04-10

---

## Context

A dental student needs to track clinical graduation requirements across 8 disciplines (Diagnosis, Operative, Peds, Perio, Endo, Prosth, Surgery, Ortho). Each discipline has a set of procedures requiring a specific number of formative and/or summative completions. Currently tracked in an Excel workbook (`Formative-Summative Tracker BLANK.xlsx`). The goal is a purpose-built web app that makes logging fast, shows progress clearly, and replaces manual spreadsheet entry.

---

## Decisions Made

| Question | Decision |
|---|---|
| Delivery format | Single `.html` file, localStorage for persistence |
| Layout | Sidebar (disciplines) + detail panel (procedure list) |
| Entry fields | Date, doctor, axiUm #, axiUm form code, notes |
| Doctor input | Dropdown of saved doctors + "add new" inline |
| Entry history | Shown per-procedure; individual entries deletable |
| Requirements | Editable via a Settings page (not front-and-center) |
| Future plan | Accounts/multi-user to be added later |

---

## Architecture

**Single HTML file** — no build step, no server, no dependencies. Open in any browser.

**Storage:** `localStorage` with a single key `swipeTracker` containing:
```json
{
  "requirements": {
    "Diagnosis": {
      "Tx Plan Basic":        { "formative": 1, "summative": 0 },
      "Tx Plan Complex":      { "formative": 4, "summative": 0 },
      "NPI":                  { "formative": 2, "summative": 1 },
      "Completion of Comp Care": { "formative": 2, "summative": 0 },
      "Emergency":            { "formative": 2, "summative": 1 },
      "Special Care":         { "formative": 4, "summative": 1 },
      "Collab w Dental Hygienist": { "formative": 2, "summative": 0 },
      "Patient Assessment":   { "formative": 0, "summative": 1 },
      "Outcome of Comp Tx & Recall": { "formative": 0, "summative": 1 },
      "Interprofessional Comm & Pt Presentation": { "formative": 0, "summative": 1 },
      "Caries Management":    { "formative": 5, "summative": 1 }
    },
    "Operative": { ... },
    "Peds": { ... },
    "Perio": { ... },
    "Endo": { ... },
    "Prosth": { ... },
    "Surgery": { ... },
    "Ortho": { ... }
  },
  "entries": [
    {
      "id": "uuid-v4",
      "discipline": "Diagnosis",
      "procedure": "NPI",
      "type": "formative",
      "date": "2026-04-10",
      "doctor": "Dr. Pullano",
      "axiUmNumber": "50100511",
      "axiUmFormCode": "",
      "notes": "Straightforward case",
      "createdAt": 1744300000
    }
  ],
  "doctors": ["Dr. Pullano", "Dr. Smith"]
}
```

**No external dependencies.** All JS and CSS inline in the file.

---

## Components

### 1. Top Navigation Bar
- App name ("Swipe Tracker") on left
- "Tracker" and "Settings" nav links in center
- Global counts ("Formatives: 14/47 | Summatives: 3/18") on right — live-computed from entries vs. requirements

### 2. Sidebar
- Lists all 8 disciplines in fixed order
- Each item shows a `done/total` badge computed from entries
- Active discipline highlighted
- Clicking switches the main panel

### 3. Main Panel — Tracker View
- **Discipline title** + overall progress bar (formatives + summatives combined)
- **Formatives section** — table of procedures requiring ≥1 formative
- **Summatives section** — table of procedures requiring ≥1 summative
- Procedures with 0 requirement in a category are omitted from that section

**Per-procedure row:**
- Procedure name
- Done count (color-coded: green = met, orange = partial, red = none)
- Needed count
- Expand chevron (▶/▼)

**Expanded row (inline log panel):**
- Lists each logged entry: date · doctor · axiUm # · notes (truncated)
- Delete button (✕) on each entry
- "+ Log [procedure] [type]" button at the bottom

### 4. Add Entry Modal
Triggered by the "+ Log" button. Fields:
- **Date** — date picker, defaults to today
- **axiUm #** — text input
- **Doctor / Instructor** — `<select>` populated from `doctors[]`; last option is "Add new doctor…" which replaces the select with a text input and saves the new name to `doctors[]`
- **axiUm Form Code** — text input (relevant for summatives, but shown for all)
- **Notes** — textarea
- **Save** / **Cancel** buttons

On save: appends entry to `entries[]`, persists to localStorage, closes modal, refreshes counts.

### 5. Settings Page
Accessed via "Settings" nav link. Contains:
- **Requirements table** — one row per discipline+procedure combination, with editable "Formative needed" and "Summative needed" number inputs
- Changes auto-save to localStorage on blur
- **Doctors list** — shows saved doctors with a delete button per entry and an "Add doctor" text input

### 6. Data Utilities (inline JS functions)
- `getState()` / `setState(data)` — read/write localStorage
- `getEntriesFor(discipline, procedure, type)` — filtered entries
- `getCountFor(discipline, procedure, type)` — length of above
- `generateId()` — simple UUID via `crypto.randomUUID()`
- `computeGlobalProgress()` — totals across all disciplines for the nav bar
- `initDefaultRequirements()` — seeds requirements from the Excel on first load if localStorage is empty

---

## Requirements Data (seeded from Excel)

All required counts seeded from `Formative-Summative Tracker BLANK.xlsx`:

| Discipline | Procedure | Form | Summ |
|---|---|---|---|
| Diagnosis | Tx Plan Basic | 1 | 0 |
| Diagnosis | Tx Plan Complex | 4 | 0 |
| Diagnosis | NPI | 2 | 1 |
| Diagnosis | Completion of Comp Care | 2 | 0 |
| Diagnosis | Emergency | 2 | 1 |
| Diagnosis | Special Care | 4 | 1 |
| Diagnosis | Collab w Dental Hygienist | 2 | 0 |
| Diagnosis | Patient Assessment | 0 | 1 |
| Diagnosis | Outcome of Comp Tx & Recall | 0 | 1 |
| Diagnosis | Interprofessional Comm & Pt Presentation | 0 | 1 |
| Diagnosis | Caries Management | 5 | 1 |
| Operative | Total Fillings | 5 | 0 |
| Operative | Class I | 1 | 1 |
| Operative | Class II | 1 | 1 |
| Operative | Class III/IV | 1 | 1 |
| Operative | Class V | 1 | 1 |
| Operative | Special Care | 4 | 1 |
| Peds | New/Recall Exam | 4 | 1 |
| Peds | Prophy | 4 | 0 |
| Peds | Fluoride | 4 | 0 |
| Peds | Bitewings (x2) | 3 | 0 |
| Peds | Xray on peds patient | 1 | 0 |
| Peds | Class I/II | 2 | 1 |
| Peds | Class III/IV/V | 2 | 0 |
| Peds | Sealant | 4 | 1 |
| Peds | Infant Oral Exam | 2 | 0 |
| Peds | Peds Case Completion | 2 | 0 |
| Peds | Complex | 1 | 0 |
| Peds | Pulp Therapy | 1 | 0 |
| Perio | Assists | 4 | 0 |
| Perio | Perio Chart | 6 | 2 |
| Perio | Perio Diagnosis | 7 | 1 |
| Perio | Prophy | 7 | 1 |
| Perio | SRP (Full) | 9 | 1 |
| Perio | Perio Surgery | 3 | 1 |
| Perio | Perio Maintenance | 5 | 1 |
| Perio | Perio Re-Eval | 5 | 1 |
| Endo | Assists | 4 | 0 |
| Endo | Case Difficulty | 5 | 0 |
| Endo | Endo Diagnosis | 5 | 0 |
| Endo | NonSurg Endo | 4 | 1 |
| Endo | Endo Recall | 2 | 1 |
| Prosth | Single Crown | 6 | 1 |
| Prosth | Fixed Partial | 1 | 1 |
| Prosth | Complete Denture | 1 | 1 |
| Prosth | Removable Partial | 1 | 1 |
| Prosth | Lab Prescription | 6 | 1 |
| Prosth | Implant Crown | 1 | 1 |
| Surgery | Presurg Case | 2 | 0 |
| Surgery | Simple/Surgical Extraction | 5 | 1 |
| Surgery | Multiple Tooth Extraction | 1 | 1 |
| Surgery | Inhalation Sedation | 2 | 1 |
| Surgery | Prescription Writing | 0 | 1 |
| Surgery | Medical Risk Assessment | 0 | 1 |
| Surgery | Local Anesthesia | 0 | 1 |
| Ortho | Consultation | 3 | 1 |
| Ortho | TMD | 0 | 1 |

---

## Visual Design

- **Color scheme:** Dark background (`#0f0f1a`), panel surfaces (`#141428`, `#1a1a2e`), accent blue (`#4a90d9` / `#6a9fd8`)
- **Status colors:** Green `#4caf50` (met), Orange `#ff9800` (partial), Red `#f44336` (none)
- **Typography:** System font stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`)
- **No external CSS frameworks** — all styles inline

---

## Error Handling

- If localStorage is unavailable (private browsing), show a banner warning that data won't persist
- Delete entry: confirm with `window.confirm()` before removing
- "Add new doctor" input: trim whitespace, ignore empty submissions, avoid duplicates

---

## Out of Scope (for now)

- User accounts / authentication (planned for future)
- Data export (CSV or JSON backup)
- Case Types sheet tracking (the Type 1–6 case requirements)
- Mobile-optimized layout
- Sync across devices
