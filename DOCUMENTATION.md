# Job Tracker — Documentation

This document explains how the tracker works under the hood: the data model, every feature, the spreadsheet and calendar file formats it produces, and how to customize it.

`index.html` is the entire application — plain HTML, CSS, and vanilla JavaScript in one file. There's no build step, no framework, and no server. Open the file (or the GitHub Pages URL) and it runs.

## Contents

- [How data is stored](#how-data-is-stored)
- [The application record](#the-application-record)
- [Feature walkthrough](#feature-walkthrough)
- [Spreadsheet export/import (CSV)](#spreadsheet-exportimport-csv)
- [Calendar invites (.ics)](#calendar-invites-ics)
- [Light/dark theming](#lightdark-theming)
- [Customizing](#customizing)
- [Limitations](#limitations)
- [File structure](#file-structure)

## How data is stored

Every application you log is kept in the browser's `localStorage`, under the key:

```
job-search-log-v2
```

as a single JSON array. There's no backend, no account, and no network request involved in saving — every change (adding a row, ticking a checklist item, editing a note) calls `save()`, which does:

```js
localStorage.setItem(STORAGE_KEY, JSON.stringify(apps));
```

and `load()` reads it back on page open. This means:

- Your data lives in **one browser, on one device**. It does not sync between Chrome and Firefox, or between your laptop and phone, even if both open the same URL.
- Clearing browser data/history for the site wipes it. There's no server copy.
- The CSV export (below) is the recommended way to back up or move data between browsers/devices.

## The application record

Each row in the tracker is one JavaScript object with this shape:

| Field | Type | Set from | Notes |
|---|---|---|---|
| `id` | string | generated | Random 8-character id, used as the row/DOM key |
| `company` | string | Add form | Required |
| `role` | string | Add form | Required |
| `date` | string (`YYYY-MM-DD`) | Add form | Date you applied; required |
| `link` | string (URL) | Add form | Optional; if set, the company name links out |
| `status` | string | Add form / status dropdown | One of `applied`, `interview`, `offer`, `rejected`, `ghosted` |
| `notes` | string | Inline notes cell | Freeform, short |
| `round` | string | Prep panel | e.g. "Phone screen", "Onsite", "Final" |
| `format` | string | Prep panel | e.g. "Phone", "Video", "Onsite", "Take-home" |
| `interviewer` | string | Prep panel | Freeform name(s) |
| `interviewDate` | string (`YYYY-MM-DD`) | Prep panel | Drives the "Next up" banner and the `.ics` download |
| `interviewTime` | string (`HH:MM`, 24h) | Prep panel | Optional; defaults to 9:00 AM in the calendar invite if left blank |
| `prepDeadline` | string (`YYYY-MM-DD`) | Prep panel | "Have prep done by" |
| `prepNotes` | string | Prep panel | Longer freeform notes, separate from the row's short `notes` |
| `prepChecklist` | array of `{ id, text, done }` | Prep panel | Seeded with 4 default items when a row is created |

New rows are seeded with a default checklist (see `DEFAULT_CHECKLIST` in the script):

1. Research the company
2. Review the job description
3. Prepare 3 questions to ask
4. Practice likely interview questions

You can rename, delete, or add to these per application — they aren't shared across rows.

## Feature walkthrough

**Add application form** — company, role, applied date, and an optional posting link. Applied date defaults to today. On submit, a new record is created with `status: "applied"` and a fresh default checklist.

**Stats bar** — five tiles (Total, Applied, Interviewing, Offers, Rejected) recomputed from the full `apps` array every time something changes. Purely derived — nothing here is stored separately.

**"Next up" banner** — appears only when at least one application has an `interviewDate` today or later. It picks the soonest one (sorted by date, then time) and shows two shortcuts: **Open prep** (jumps into that row's prep panel) and **Add to calendar** (downloads the `.ics` directly, without opening the panel).

**Filter chips + search** — the status chips (`All`, `Applied`, `Interview`, …) and the search box both filter the same in-memory list before rendering rows; they don't change what's stored, only what's currently visible. Search matches company or role, case-insensitive.

**Status dropdown** — each row's status pill is a `<select>` styled to look like a colored chip. Changing it updates `status` and immediately re-renders the stats and banner.

**Inline notes** — a plain text input embedded in the row. Saves on blur/change (not on every keystroke), matching how a spreadsheet cell behaves.

**Prep panel** (opened via the "Prep" button, which also shows a live `done/total` fraction like `2/4`) contains:
- *Interview details* — round, format, interviewer, date, and time.
- *Download calendar invite (.ics)* — enabled once a date is set (see below).
- *Prep deadline* — a single date field for when you want prep finished.
- *Prep checklist* — checkboxes with editable text and a remove button per item, plus an "Add a prep task…" input. Checking an item strikes through its text.
- *Notes* — a longer freeform textarea, distinct from the row's inline note.

All fields in the panel save on change; there's no separate "save" button, and closing the panel (✕, clicking outside, or Escape) just re-renders the table/banner to reflect what changed.

**Delete** — the ✕ button on the far right of a row removes that application entirely, with no confirmation (undo is "add it again"). There's no soft-delete or trash.

## Spreadsheet export/import (CSV)

The **Export spreadsheet (.csv)** button serializes every field listed in `CSV_COLUMNS` (all the fields in the table above except `id`) into a standard CSV file, one row per application, with a header row. It opens directly in Excel, Google Sheets, or Numbers.

A CSV cell is quoted (and internal quotes doubled) whenever it contains a comma, quote character, or newline — the same escaping rule spreadsheets themselves use, implemented by hand in `csvEscape()`/`parseCsv()` rather than pulling in a charting/spreadsheet library. This was a deliberate choice: generating a true binary `.xlsx` in-browser needs an external library, and the sandboxed environment this tool was originally built in blocks loading one from a CDN. CSV avoids that dependency entirely while still round-tripping cleanly through Excel and Sheets.

**Checklist serialization** — since a checklist is a list of `{text, done}` pairs and CSV cells are flat text, each item is packed as `text|0` or `text|1` (0/1 for done), and items are joined with `;;`. For example, a two-item checklist serializes to:

```
Research the company|1;;Review the job description|0
```

Any literal `|` in your own checklist text gets replaced with `/` on export so it can't be confused with the delimiter.

**Import** — reading the file back parses the CSV (handling quoted fields, embedded commas, and embedded newlines), rebuilds each row's object from the header, regenerates checklist items with new internal ids, and defaults any missing/invalid status back to `applied`. You're then asked to confirm:

- **OK** — replaces your current log entirely with the imported rows.
- **Cancel** — appends the imported rows alongside what's already there.

This makes CSV both your backup format and your migration path between browsers or devices — export from one, import on the other.

## Calendar invites (.ics)

Once a row has an `interviewDate` (time is optional, defaulting to 9:00 AM), the **Download calendar invite (.ics)** button builds a standard [iCalendar](https://www.rfc-editor.org/rfc/rfc5545) `VEVENT`:

- **Summary**: `Interview: {role} at {company}`
- **Start/end**: the interview date/time, with a 1-hour duration
- **Description**: round and interviewer, plus your prep notes if you've written any
- **Location**: the format field (Phone/Video/Onsite/Take-home), if set

The file is generated as a `Blob` and downloaded via a temporary `<a download>` link — nothing is sent over the network. Times are written as "floating" local time (no timezone or `Z` suffix), so whatever calendar app imports it will schedule it in that app's own local timezone — the simplest behavior for a single-user tool where the interview time you type in is the time in your own timezone.

There's no live sync with Google Calendar or any other calendar service: this repo has no server component and no OAuth flow, so each interview is a manual one-click download-and-import rather than an automatic two-way sync.

## Light/dark theming

Colors are defined as CSS custom properties on `:root`, redefined under `@media (prefers-color-scheme: dark)` to follow the OS/browser theme automatically, and redefined again under explicit `:root[data-theme="dark"]` / `:root[data-theme="light"]` selectors so a manual theme toggle (e.g., if this is embedded somewhere that stamps that attribute) overrides the OS preference. Every component reads color through these tokens (`--bg`, `--ink`, `--accent`, per-status `--applied`/`--interview`/`--offer`/`--rejected`/`--ghosted` pairs, etc.) rather than hardcoding hex values, so retheming means editing the token block, not hunting through component styles.

## Customizing

Everything lives in `index.html`, so customizing means editing that file directly:

- **Add a status** — add an entry to the `STATUSES` array (`{ id, label }`) near the top of the `<script>`, then add matching `--yourstatus` / `--yourstatus-bg` CSS variables (light, dark, and both explicit theme blocks) and a `.status-select.yourstatus` rule.
- **Change the default checklist** — edit the `DEFAULT_CHECKLIST` array of strings; it only affects newly created applications, not existing ones.
- **Change colors/fonts** — edit the `:root` custom properties and the font stacks set on `body`/headings/mono elements at the top of the `<style>` block.
- **Change the calendar event duration** — it's hardcoded to 1 hour in `downloadIcs()`; adjust the `60 * 60 * 1000` millisecond offset.

## Limitations

- **No cross-device sync.** Data is per-browser `localStorage`. Use CSV export/import to move it.
- **No accounts, no backend.** Nothing you enter ever leaves your browser except when you explicitly download a `.csv` or `.ics` file.
- **CSV, not a styled workbook.** Export is plain CSV — no cell formatting, formulas, or multiple sheets. It's built for portability, not presentation.
- **No live Google Calendar (or any) sync.** `.ics` download + manual import is the whole integration.
- **Single user.** There's no multi-user or collaboration model — it's one array in one browser.

## File structure

```
Job-Tracker/
├── index.html         the entire app: markup, styles, and script
├── README.md           quick start + GitHub Pages instructions
└── DOCUMENTATION.md    this file
```
