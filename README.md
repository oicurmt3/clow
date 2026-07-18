# Clow

**Clow** (“create + flow”) is a browser-based tool for building and running **Flows** — ordered decision steps that steer people toward relevant, robust evidence against key questions.

Use it for live investigation runbooks or education/training rehearsals. Same engine; different intent.

**Live concept:** [https://sandgroper.net/clow](https://sandgroper.net/clow)

---

## What it does

- **Library** — create, duplicate, import/export Flows; resume in-progress runs; view completed runs
- **Builder** — author question and outcome steps, answer controls, and IF/THEN AND/OR routing rules
- **Runner** — walk a path with Entity + case number, matched-rule explanations, evidence action checklists, Print / Export
- **Training** — standalone guide (`training.html`) covering concepts, saving, rules, and practice

There is **no server account**. Everything stays in the browser until you export files.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | Full app (UI + logic) |
| `training.html` | Training guide (hash chapters) |
| `DEVELOPER_HANDOVER.md` | Architecture and function map for maintainers |

No build step, no `package.json`, no backend.

---

## Quick start

1. Open `index.html` in a modern browser (or deploy both HTML files to any HTTPS static host).
2. Use **Reset demo** (on the demo card) to load the sample Flow, or **New Flow** to author your own.
3. **Open training** from the library (or open `training.html`) for the full guide.

**Deploy (e.g. cPanel):** upload `index.html` and `training.html` to the same folder. Serve over HTTPS so fonts and external image URLs work cleanly.

---

## Data & backups

| Storage | Contents |
|---------|----------|
| `localStorage` key `clow.v1` | All Flows and runs |
| `localStorage` key `clow.lastRunner` | Last Entity + case number (prefill) |

**Export all** downloads a full backup JSON. Export a single Flow from a card or the builder. Export a path as JSON or HTML from the runner (or Export JSON from a library run card).

Clearing site data wipes the library. Export before browser cleanup or when moving machines. Another browser/profile will not see your data until you Import a backup.

Details: training chapter [Saving & backups](training.html#saving) and `DEVELOPER_HANDOVER.md`.

---

## Core ideas

- **Flow** — named set of steps you build once and run many times
- **Question** — choice control + ordered IF rules + ELSE (default next)
- **Outcome** — checkpoint (continue) or ending (end here) with guidance, links, image, evidence actions
- **Run** — one walk through a Flow; answers and route reasons are saved
- **Entity + case number** — required on the first step; name the run for library cards and exports

Routing: rules top → bottom; within a rule AND = all true, OR = any true; first match wins; otherwise ELSE.

---

## Export file types

| Type | Typical file | Contents |
|------|--------------|----------|
| Backup | `clow-backup.json` | Every Flow + every run |
| Flow | `clow-flow-….json` | One Flow definition |
| Path | `clow-path-….json` / `.html` | One run’s answers, reasons, actions |

Import backups and path JSON from the **library**. Import Flow JSON from the library or the **builder** (Replace draft / Add as new).

---

## Browser support

Modern evergreen browsers with `localStorage` and ES2019+ JavaScript. Designed as a light-grey static webapp for HTTPS hosting.

---

## License / ownership

Add your preferred license and contact here when publishing the repository.
