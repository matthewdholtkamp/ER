# ER Dashboard — Parsing Improvement Plan (Phase 1)

Handoff spec for Codex / Antigravity. Single-file app: `index.html` (~1,416 lines, HTML+CSS+vanilla JS). No backend. XLSX parsed in-browser via JSZip + DOMParser. Charts via Chart.js.

**Hard constraint:** Do NOT change how Excel files are authored. All legacy + future data keeps its current structure. All improvements must work against the existing format.

---

## Scope

Phase 1 (this plan):
1. Normalize inconsistent military unit names into one canonical unit.
2. Add Brigade → Battalion → Company hierarchy + slicer (drill-down filter).
3. Add time-range filtering (e.g. last 6 months, last week).

Phase 2 (DO NOT build yet): integrate user's "Band-Aid Six" AI agent persona to search/answer questions about the repo.

## Inputs still owed by user (blockers, flag before merging)
- (a) Template Excel files currently in use (to validate parsing against real variants).
- (b) Brigade → unit mapping (not in the Excel; required for the brigade slicer level).
- (c) Time-series data spanning months (to test time-range filtering).

Build the code so (b) and (c) plug in via a config object + already-parsed dates; do not hardcode.

---

## Task 1 — Unit normalization

### Current code (root cause)
`index.html:306`
```js
const UNIT_REGEX = /^([A-Za-z])[\/\-]?(\d{2,4}|FTU|TU|RHU)$/i;
function parseUnit(text) {
  if (!text) return null;
  const m = String(text).trim().match(UNIT_REGEX);
  if (!m) return null;
  return { company: m[1].toUpperCase(), battalion: m[2].toUpperCase() };
}
```
Only matches compact forms: `B/43`, `B-43`, `B43`, or codes `FTU/TU/RHU`.
**Fails on:** `43rd`, `43 rd`, `43` (battalion only), `bravo 43` (spelled-out company), space-separated forms, company-after-battalion ordering.

Also used as a fallback on the last name token at `index.html:692-699`, and `cleanName` strips the trailing unit with a matching narrow regex at `index.html:705`.

### Required behavior
Rewrite `parseUnit` to return `{ company, battalion }` (either may be empty string if absent) by handling, case-insensitively, any combination of:
- Battalion as 2–4 digits, with optional ordinal suffix `st|nd|rd|th`, with or without a space: `43`, `43rd`, `43 rd`, `43RD`.
- Special battalion codes: `FTU`, `TU`, `RHU`.
- Company as a single letter (`A`–`F` etc.) OR spelled-out NATO phonetic (`alpha→A, bravo→B, charlie→C, delta→D, echo→E, foxtrot→F`, etc. — include full A–Z phonetic table).
- Either order: `B/43`, `B 43`, `bravo 43`, `43 bravo`, `43/B`.
- Battalion-only or company-only inputs.

### Implementation guidance
1. Add a phonetic map constant near line 306:
   ```js
   const PHONETIC = {alpha:'A',bravo:'B',charlie:'C',delta:'D',echo:'E',foxtrot:'F',golf:'G',hotel:'H',india:'I',juliet:'J',kilo:'K',lima:'L',mike:'M',november:'N',oscar:'O',papa:'P',quebec:'Q',romeo:'R',sierra:'S',tango:'T',uniform:'U',victor:'V',whiskey:'W',xray:'X',yankee:'Y',zulu:'Z'};
   ```
2. Rewrite `parseUnit` as a tokenizer, not one regex:
   - Lowercase + trim. Strip `/`, `-` to spaces. Collapse whitespace.
   - Token classifiers:
     - `normBattalion(tok)`: matches `^(\d{2,4})(st|nd|rd|th)?$` → digits; also `^(ftu|tu|rhu)$` → uppercased.
     - `normCompany(tok)`: single `[a-z]` → upper; or `PHONETIC[tok]`.
   - Handle the glued case `43rd`/`b43` by also testing each token against `^([a-z])(\d{2,4})(st|nd|rd|th)?$` and `^(\d{2,4})(st|nd|rd|th)?([a-z])$`.
   - Return `{company, battalion}` from whatever was found; `null` only if neither found.
3. Canonical key helper for grouping/dedupe:
   ```js
   function unitKey(u){ return u ? `${u.battalion||'?'}-${u.company||'?'}` : ''; }
   ```
   Use this everywhere a unit is grouped (the battalion/company chart `renderBnCompanyChart` at `index.html:1283`).
4. Update `cleanName` (`index.html:705`) to strip the broader set of trailing unit tokens (ordinals, phonetic words, space-separated forms), not just `\s+[A-Za-z][\/\-]?\d{2,4}\s*$`.
5. Keep `parseSheet` flow (`index.html:692-699`): try `unitCell` first, then fallback to name token(s) — but pass the last 1–2 tokens to the new parser so `bravo 43` at the end of a name is caught.

### Acceptance
All of these resolve to battalion `43`, company `B`:
`B/43`, `B-43`, `B43`, `43/B`, `bravo 43`, `43 bravo`, `B 43rd`, `43 rd B`.
`43` alone → battalion `43`, company empty (still groups by battalion). No regressions on existing `FTU/TU/RHU`.

---

## Task 2 — Brigade → Battalion → Company slicer

### Data model
Add a config constant. **Three confirmed brigades** (from user's org chart image): `3rd Chemical Brigade`, `14th MP Brigade`, `1st Engineer Brigade`. The battalion→brigade rows below are a scaffold — the exact battalion codes under each brigade must be transcribed from the user's image before this map is complete. Keys are the canonical battalion value produced by `parseUnit` (digits like `43`, or codes `FTU/TU/RHU`).
```js
// canonical battalion code -> brigade label
const BRIGADE_MAP = {
  // --- 3rd Chemical Brigade ---
  // "<bn>": "3rd Chemical Brigade",
  // --- 14th MP Brigade ---
  // "<bn>": "14th MP Brigade",
  // --- 1st Engineer Brigade ---
  // "<bn>": "1st Engineer Brigade",
};
function brigadeFor(battalion){ return BRIGADE_MAP[battalion] || 'Unassigned'; }
```
Attach `brigade` to each patient at parse time in `parseSheet` (the patient object at `index.html:708`).

**Blocker:** the specific battalion lists under each brigade are in the user's image and have NOT yet been transcribed. Populate the three groups above from that image (or from the example Excel files the user will send) before the brigade slicer can map real units. Until then the map stays empty and everything falls under "Unassigned" (see Acceptance).

### UI
- Add three dependent `<select>` slicers above the battalion/company chart: Brigade → Battalion → Company.
- Selecting a brigade filters the battalion options to that brigade; selecting a battalion filters company options; "All" at each level.
- Filtered patient set feeds `renderBnCompanyChart` (`index.html:1283`) and the daily/patient tables (`renderDailyTable` :1359, `renderPatientTable` :1372).
- Keep a single `getFilteredPatients()` helper that applies brigade/battalion/company + time-range (Task 3) so all renderers share one filtered array.

### Acceptance
- With `BRIGADE_MAP` empty, everything shows under "Unassigned" and nothing breaks.
- Selecting Brigade → Battalion → Company narrows the chart + tables consistently.

---

## Task 3 — Time-range filtering

Each parsed file already carries a `date` (`parseDateFromFilename` :254; sheet result includes `date` :731). Patients inherit their file's date.

### UI
- Add a range control: presets (Last 7 days, Last 30 days, Last 6 months, All) + optional custom start/end date inputs.
- Default preset = current behavior of the "Last 7 Days" chart so nothing regresses.

### Logic
- In `getFilteredPatients()`, filter by `patient date` within the selected window (computed relative to the most recent date in the dataset, not `today`, so historical drops work).
- Re-render all charts/tables from the filtered set.

### Acceptance
- Switching ranges updates the battalion/company chart, daily table, patient table.
- "All" shows everything; presets bound correctly off the dataset's max date.

---

## Execution rules (token efficiency for the implementing agent)
- Work in this order: Task 1 → verify → commit; Task 2 → verify → commit; Task 3 → verify → commit.
- Use targeted line-range reads + Grep, not whole-file reads. Key anchors: `UNIT_REGEX` :306, `parseUnit` :308, `parseSheet` :633, patient object :708, `cleanName` :705, `renderBnCompanyChart` :1283, `renderDailyTable` :1359, `renderPatientTable` :1372, `clearAll` :1389, `exportCSV` :1397.
- After each task, open `index.html` in a browser, drop a sample workbook, confirm charts/tables render and no console errors.
- Do not introduce build tooling, frameworks, or a backend. Stay single-file.
- Do not alter the Excel-authoring expectations.

## Out of scope (Phase 2)
"Band-Aid Six" AI agent persona to search/answer questions about this repo. Not part of this plan.
