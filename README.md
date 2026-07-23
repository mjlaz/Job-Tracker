# Job Tracker

A single-file job search tracker. Log applications, track status (Applied → Interview → Offer / Rejected / Ghosted), keep a prep checklist per interview, and download calendar invites for interview dates.

## Use it

Open `index.html` in any browser. Everything is saved to that browser's local storage — no account, no server.

To host it live via GitHub Pages: **Settings → Pages → Deploy from branch → main → / (root)**. It'll be served at `https://mjlaz.github.io/Job-Tracker/`.

## Features

- Application log with status pills and a running stats summary
- Per-interview prep checklist, round/format/interviewer notes, and a prep deadline
- Calendar invite (`.ics`) download for scheduled interviews
- Export/import as CSV — opens directly in Excel, Google Sheets, or Numbers

## How it works

See [DOCUMENTATION.md](DOCUMENTATION.md) for the data model, the CSV/`.ics` file formats, theming, and how to customize it.
