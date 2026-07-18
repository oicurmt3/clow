# Clow — Developer handover

This document explains how the Clow codebase is structured, how data is stored, and what the major functions do. Use it when taking over maintenance or extending the app.

**Product intent:** Flows that obtain relevant, robust evidence against key questions — for investigation or training.

**Repo shape:** two static HTML files, no build toolchain, no backend.

---

## 1. Repository layout

| File | Role |
|------|------|
| `index.html` | Entire product SPA: CSS design system, shell DOM, library / builder / runner logic |
| `training.html` | Standalone training guide; hash-routed chapters; does not write app storage |
| `README.md` | GitHub-facing overview for users and deployers |
| `DEVELOPER_HANDOVER.md` | This file |

**External dependencies (CDN):**

- Google Fonts: DM Sans, Source Serif 4
- Material Symbols Outlined (builder icons)
- Demo Flow may reference external image URLs (e.g. picsum); Flows store URL strings only

**Hosting:** upload both HTML files to the same origin (cPanel / any static HTTPS host). Relative links: `index.html` ↔ `training.html`.

---

## 2. Architecture

```
┌─────────────────────────────────────────┐
│  index.html (IIFE)                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐│
│  │ Library │→ │ Builder │→ │ Runner  ││
│  └────┬────┘  └────┬────┘  └────┬────┘│
│       │            │ Save       │ auto│
│       └────────────┴────────────┘     │
│                    ↓                  │
│            localStorage clow.v1       │
│            localStorage clow.lastRunner│
└─────────────────────────────────────────┘
         training.html (docs only)
```

- **View router:** in-memory `app.view` ∈ `"library" | "builder" | "runner"`. `navigate(view, opts)` then `render()`.
- **Builder draft:** `app.draft` is a deep clone of a Flow. The library shows the last **saved** version until the user presses Save.
- **Runs:** created and updated immediately via `saveRun` (Next / Back / action marks / complete).
- **No framework:** vanilla DOM; HTML strings + `innerHTML` + event binding after each render.
- **Shell DOM:** sticky header (`#brandHome`, Training link, `#headerActions`), `#app`, `#modalRoot`, `#toastRoot`.

### App UI state (`app`)

| Field | Meaning |
|-------|---------|
| `view` | Current screen |
| `workflowId` | Flow being edited or related to run |
| `runId` | Active run |
| `selectedNodeId` | Builder step selection |
| `draft` | Builder in-memory Flow (or `null`) |
| `runnerValue` | Current answer control value |
| `showPresets` | Quick-add presets dropdown open |
| `testMode` | Test run started from builder (leave → builder) |

---

## 3. Persistence

### Keys

| Constant | Value | Contents |
|----------|-------|----------|
| `STORAGE_KEY` | `"clow.v1"` | `{ workflows, runs, version: 1 }` |
| `LAST_RUNNER_KEY` | `"clow.lastRunner"` | `{ userName, caseNumber }` (Entity + case prefs) |
| `DEMO_ID` | `"wf_demo_clow"` | Built-in demo Flow id |

### Load / save lifecycle

| Function | Behaviour |
|----------|-----------|
| `defaultState()` | Demo Flow only; empty `runs` |
| `loadState()` | Parse `clow.v1`; migrate workflows; ensure demo exists; rewrite storage; on error → default |
| `saveState(state)` | `JSON.stringify` into `localStorage` |
| `saveWorkflow(wf)` | Migrate, stamp `updatedAt`, upsert, `saveState` |
| `deleteWorkflow(id)` | Remove Flow **and** all runs with that `workflowId` |
| `saveRun` / `deleteRun` | Upsert / remove run, then `saveState` |
| `resetDemo()` | Replace demo Flow with `createDemoWorkflow()`; delete demo runs only |
| `loadLastRunner` / `saveLastRunner` | Prefill Entity + case on new runs |

**Important:** Builder edits are **not** in `clow.v1` until Save. Leaving the builder with unsaved changes shows a confirm modal.

---

## 4. Data model

### Workflow (Flow)

```js
{
  id, name, tags[], isDemo,
  createdAt, updatedAt,
  startNodeId,
  stepOrder[],          // sidebar / list order (not routing)
  nodes: { [nodeId]: Node }
}
```

### Question node

```js
{
  id, title, kind: "question",
  prompt, guidance,
  links: [{ label, url }],
  fieldType: "radio" | "checkbox" | "toggle" | "cycle" | "select" | "slider",
  options: [{ value, label }],
  slider?: { min, max, step, bands: [{ max, label, routeKey }] },
  defaultValue,
  defaultRoute,         // ELSE → node id
  rules: [ Rule ]
}
```

### Outcome node

```js
{
  id, title, kind: "outcome",
  prompt, guidance, links, imageUrl,
  endHere,              // true = terminal; false = checkpoint
  actions: [ string ],  // evidence / action lines
  next                  // next node id when checkpoint
}
```

Legacy: `kind: "info"` → migrated to continuing outcome. Legacy `routes` map → `rules[]`.

### Rule / condition

```js
Rule = {
  id,
  join: "and" | "or",
  conditions: [ Condition ],
  then: nodeId
}
Condition = {
  stepId,                           // current or prior question
  op: "eq" | "neq" | "includes" | "excludes" | "in_band",
  value
}
```

### Run

```js
{
  id, workflowId,
  userName,             // UI label: Entity
  caseNumber,
  name,                 // composed display title
  status: "in_progress" | "completed",
  createdAt, updatedAt, completedAt?,
  isTest,
  currentNodeId,
  answers: [ Answer ],
  actions: [],          // snapshot from terminal outcome at complete
  actionMarks: { "[nodeId]:[index]": "null" | "done" | "na" }
}
```

### Answer (path step)

```js
{
  nodeId, title, prompt,
  value,                // control value, or "checkpoint"
  display,              // human label
  routeReason           // "Matched rule N: …" or "ELSE → …"
}
```

---

## 5. Routing engine (IF / THEN / ELSE)

**Entry point for the runner:** `resolveRouteDetail(wf, node, run, value)` → `{ nextId, reason }`.

Thin wrapper: `resolveRoute` → `nextId` only.

### Algorithm

1. **Outcome**
   - Terminal (`isTerminalOutcome` / `endHere !== false`) → `{ nextId: null }`
   - Checkpoint → `{ nextId: node.next, reason: "Continue → …" }`
2. **Question**
   - Build answer map: `buildAnswersByStep(run, node.id, value)` (prior answers + current)
   - Walk `node.rules` in order
   - First rule where `evalRule` is true and `then` is set wins → `formatMatchedRuleReason`
   - Else → `node.defaultRoute` with reason `ELSE → …`

### Condition evaluation (`evalCondition`)

| Op | Use |
|----|-----|
| `eq` / `neq` | Scalar answers (radio, toggle, cycle, select) |
| `includes` / `excludes` | Checkbox arrays |
| `in_band` | Slider: `sliderBand(step, num).routeKey === cond.value` |

Missing answer for `stepId` → condition false.

### Rule join (`evalRule`)

- `"or"` → `some(conditions)`
- default `"and"` → `every(conditions)`
- Empty conditions → false

### Validation (`validateWorkflow`)

Checks (among others): start exists, ELSE/THEN targets, empty rules, orphan condition values, reachability (BFS), at least one terminal outcome. Used before Run (non-test) and by Test run / Validate UI.

---

## 6. Cascade rename (option / band values)

When an option `value` or slider band `routeKey` changes, rules across the Flow that reference the old value are updated:

| Function | Role |
|----------|------|
| `cascadeAnswerValue(wf, stepId, oldVal, newVal)` | Rewrite matching condition values |
| `setOptionValue` / `setBandRouteKey` | Apply new slug + cascade |
| `removeAnswerValueRefs` | Clear refs when an option is deleted |
| `countAnswerValueRefs` / `conditionValueExists` | Orphan detection for red highlight + Validate |

Auto-style values are composed via `composeOptionValue` / `slugifyOptionPart` / `uniqueOptionValue`.

---

## 7. Import / export envelopes

| `type` | Payload | Produced by | Consumed by |
|--------|---------|-------------|-------------|
| `clow-backup` | `workflows`, `runs`, `exportedAt` | Library → Export all | Library → Import (merge by id) |
| `clow-flow` | `workflow` | Card / builder Export | Library Import; Builder Import |
| `clow-workflow` | alias of flow | accepted on import | same |
| bare Flow | `{ id, nodes, startNodeId }` | — | `extractImportedFlow` |
| `clow-run` | run + metadata | Runner Export JSON | Library Import |

**Helpers:**

- `extractImportedFlow`, `normalizeImportedFlow`, `importFlowAsNew`, `replaceBuilderDraftWithFlow`
- `promptBuilderFlowImport` — modal: Add as new vs Replace draft
- `importData` — library import dispatcher
- `buildRunExportPayload`, `buildRunReportHtml`, `exportRunJson`, `exportRunHtml`
- `downloadJson`, `downloadText`

Builder Import rejects backups and run exports (toast directs user to Library).

---

## 8. Module map — `index.html`

Line numbers drift as the file grows; search for function names or section comments.

### 8.1 Helpers

`uid`, `now`, `fmtDate`, `fmtRunName`, `loadLastRunner`, `saveLastRunner`, `composeRunName`, `runDisplayTitle`, `escapeHtml`, `downloadJson`, `downloadText`, `deepClone`

### 8.2 Seed Flows

| Function | Role |
|----------|------|
| `createDemoWorkflow()` | Large multi-branch demo (`DEMO_ID`, `isDemo: true`) — all controls, AND/OR, prior-step rules, checkpoints, endings |
| `createBlankWorkflow(name)` | Minimal starter: one question + one terminal outcome |

### 8.3 State CRUD

`defaultState`, `loadState`, `saveState`, `getWorkflows`, `getWorkflow`, `saveWorkflow`, `deleteWorkflow`, `getRuns`, `getRun`, `saveRun`, `deleteRun`, `resetDemo`

### 8.4 Nodes, migration, validation, routing

`nodeList`, `outcomeNodes`, `questionNodes`, `migrateInfoToOutcome`, `migrateWorkflowInfoNodes`, `createRule`, `createCondition`, `migrateQuestionRoutesToRules`, `migrateWorkflow`, `ensureQuestionRules`, `questionRouteTargets`, `computeBfsStepIds`, `ensureStepOrder`, `getOrderedSteps`, `isTerminalOutcome`, `normalizeOutcome`, `nodeLabel`, `optionLabel`, `sliderBand`, `buildAnswersByStep`, `evalCondition`, `evalRule`, `conditionOpLabel`, `formatConditionText`, `formatMatchedRuleReason`, **`resolveRouteDetail`**, `resolveRoute`, `pathAnswerHtml`, `pathListItemHtml`, `estimateRemaining`, **`validateWorkflow`**

### 8.5 Chrome

`toast`, `closeModal`, `showModal`, `setHeaderActions`, `navigate`, brand home → library

### 8.6 Library

| Function | Role |
|----------|------|
| `libraryRunCardHtml(r)` | In-progress / completed run card markup |
| `renderLibrary()` | Flows list, Import / Export all / New; run sections; Reset demo confirm; open/export/delete runs |

### 8.7 Options / cascade

`slug`, `slugifyOptionPart`, `composeOptionValue`, `uniqueOptionValue`, `cascadeAnswerValue`, `removeAnswerValueRefs`, `countAnswerValueRefs`, `conditionValueExists`, `setOptionValue`, `setBandRouteKey`

### 8.8 Import / duplicate

`extractImportedFlow`, `normalizeImportedFlow`, `importFlowAsNew`, `replaceBuilderDraftWithFlow`, `promptBuilderFlowImport`, `readJsonFile`, `importData`, `duplicateWorkflow`

### 8.9 Builder

| Function | Role |
|----------|------|
| `openBuilder` | Clone Flow into `app.draft`, migrate, navigate |
| `renderBuilder` | Header actions (Save, Validate, Test, Import/Export), step list, node editor |
| `addStep` / `addPreset` | New steps / quick-add templates |
| `renderNodeEditor` | Question vs outcome editor UI |
| `renderRulesEditorHtml` / `bindRulesEditor` | IF/THEN rule UI |
| `bindRuleDragDrop` / `bindOptionDragDrop` / `bindStepDragDrop` | Reorder with drop lines |
| `convertKind` | Question ↔ outcome conversion |
| `routePreview` | Human summary of rules + ELSE |
| `targetOptionsHtml`, `questionStepOptionsHtml`, `conditionOpsForStep`, `conditionValueOptionsHtml` | Select helpers |
| `syncStepListTitle` | Keep sidebar titles in sync while typing |

**Save:** `saveWorkflow(deepClone(wf))` from draft.  
**Test run:** validate → `app.testMode = true` → `startRun(id, { test: true })`.  
**Leave:** warn if draft dirty vs saved (modal).

### 8.10 Runner

| Function | Role |
|----------|------|
| `startRun` | Validate (unless test); create in-progress run; open runner |
| `applyRunMetaFromInputs` | Require Entity + case on first Next; update `name` + lastRunner |
| `actionMarkKey`, `getActionMark`, `cycleActionMark`, … | To do / Done / N/A cycles |
| `pathExportBasename`, `buildRunExportPayload`, `collectRunActionRows`, `buildRunReportHtml` | Export builders |
| `exportRunJson` / `exportRunHtml` / `bindRunExportButtons` | Downloads |
| `openRunner` | Set `runnerValue`, navigate |
| `initRunnerValue` | Default control value from node |
| `renderRunner` | Progress chrome; Entity fields if `answers.length === 0`; step UI or end screen |
| `mountControl` | Radio / checkbox / toggle / cycle / select / slider |
| **`runnerNext`** | Meta check → checkpoint or resolve route → push answer → advance |
| **`runnerBack`** | Pop last answer; reopen prior step |
| **`renderEndScreen`** | Path, actions, Print, exports |

**Entity / case:** shown at top of first step only (`run.answers.length === 0`). Field id `runUserName` stores Entity (`userName` in data).

### 8.11 Boot

`openDropdown` / `closeAllDropdowns`, `render()` switch on `app.view`, initial `render()`.

---

## 9. Training page (`training.html`)

### Chapter list (`TRAIN_PAGES`)

| id | Group | Title |
|----|-------|-------|
| `overview` | Start | Welcome |
| `concepts` | Start | Core concepts |
| `library` | App tour | Library & data |
| `saving` | App tour | Saving & backups |
| `builder` | App tour | Builder basics |
| `controls` | App tour | Answer controls |
| `rules` | App tour | IF / THEN rules |
| `runner` | App tour | Running a path |
| `investigate` | Build for purpose | Investigation Flows |
| `educate` | Build for purpose | Education & training |
| `practice` | Practice | Hands-on practice |
| `reference` | Practice | Quick reference |

### Routing

- `currentPageId()` — `location.hash` without `#`; unknown → `overview`
- `goPage(id)` — set hash
- `render()` — sidebar from `TRAIN_PAGES` (grouped), `trainContent(id)`, prev/next
- `hashchange` → `render`
- Default: `#overview` if no hash
- Content is HTML string templates inside `trainContent` — **not** stored in `localStorage`

Update training when you change user-facing behaviour (Entity, completed runs, export types, Save semantics).

---

## 10. UI / design conventions

- Light grey theme via CSS variables (`--bg0`, `--primary`, etc.) in `:root`
- Fonts: Source Serif 4 (display), DM Sans (UI)
- Button classes: `btn-primary`, `btn-secondary`, `btn-neutral`, `btn-ghost`, `btn-danger`, sizes `btn-sm` / `btn-lg`
- Prefer matching existing patterns when adding UI
- Print styles exist for runner / training

---

## 11. Common change recipes

### Add a new answer control type

1. Extend `fieldType` handling in `renderNodeEditor` (builder options UI)
2. Handle in `mountControl` (runner)
3. Ensure `initRunnerValue` / `optionLabel` / `evalCondition` ops make sense
4. Update demo and training `#controls` if user-facing

### Add a validation rule

Extend `validateWorkflow`; surface via Validate modal and Ready/issue tags on library cards.

### Change run naming

`composeRunName` / `runDisplayTitle` / `applyRunMetaFromInputs` / library card meta.

### Change export shape

Bump or document `version` carefully; keep `importData` / `extractImportedFlow` backward compatible.

### Replace the demo

Edit `createDemoWorkflow()`. Users must **Reset demo** to reload an existing install (storage keeps the old demo until reset).

---

## 12. Mental model for maintainers

1. **Library** = Flows + runs list + full backup I/O  
2. **Builder** = draft edit + rules + Save; Test run validates then starts a marked run  
3. **Runner** = Entity/case → answer → `resolveRouteDetail` → path → terminal end screen  
4. **Training** = documentation only; keep it aligned with product behaviour  

---

## 13. Constants quick reference

| Name | Value / notes |
|------|----------------|
| `STORAGE_KEY` | `"clow.v1"` |
| `DEMO_ID` | `"wf_demo_clow"` |
| `LAST_RUNNER_KEY` | `"clow.lastRunner"` |
| `ACTION_STATUSES` | `null`, `done`, `na` (UI: To do / Done / N/A) |
| Export types | `clow-backup`, `clow-flow`, `clow-run` |
| Toast duration | ~2600 ms |
| UID helper | `uid("wf"|"n"|"r"|"run"|…)` |

---

## 14. Out of scope (intentionally)

- Server sync, auth, multi-user collaboration
- Uploaded binary assets (images are URLs)
- Free-text answer fields (choice-based controls only)
- Build bundler / TypeScript / framework rewrite (unless you choose to migrate)

When adding scope, prefer keeping the single-file deploy story unless the product requirements force a split.
