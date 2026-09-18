# Ottawa Remediation — Daily Report

**Live at `ottawa-daily.mahonryvazquez.workers.dev`** (Cloudflare Workers).
That bare hostname — no scheme, no trailing slash — is what goes in Firebase →
Authentication → Settings → **Authorized domains**, alongside Murch's. Without
it, crews still file reports perfectly well and only supervisor sign-in fails,
with `auth/unauthorized-domain`.

Field reporting app for the Ottawa, Illinois module remediation. Five screens,
Spanish and English, built to be filled in on a phone at the end of a shift.

Built from the Murch app. The infrastructure is unchanged and proven — outbox,
separate sheet queue, service worker, supervisor screen, email-link sign-in.
What changed is the domain layer:

* clock is **`America/Chicago`**, not `America/Detroit`
* **five fixed scopes** (LOTO, MC4, modules removed, modules installed, harness)
  plus a truck count, instead of a twenty-line takeoff behind a picker
* **no hour splitting at all.** Murch had to spread the day across takeoff lines
  because each earned at its own rate. Ottawa's tracker carries one total
  man-hours figure per report and derives everything from the baseline, so
  men × hours is the whole story
* crew is one of **eight named work streams**, not a number 01–15
* Firestore collection is **`ottawa_reports`** in the same Firebase project
* the block screen is **skipped entirely** while `BLOCKS` is empty

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app. All CSS and JS inline, no build step. |
| `manifest.json` | Name, icon and fullscreen behaviour when installed. |
| `sw.js` | Service worker — offline cache, versioned off `BUILD`. |
| `icons/` | App icons built from the United Services mark. |

## Releasing a change

The version string lives in **three** places and they move together:

1. `index.html` → `var BUILD = "OTT 2";`
2. `index.html` → `var BUILD_DATE = "…";`
3. `sw.js` → `var BUILD = "OTT 2";`

`sw.js` is the one that matters: the cache name is built from it, so bumping it
is what tells every phone to throw the old app away. Forget it and phones serve
the previous version indefinitely, which looks exactly like the update failing.

`BUILD_DATE` is the date at the **project site** (America/Chicago).

## Adding the blocks

`BLOCKS` at the top of `index.html` is an empty array. Fill it —

```js
var BLOCKS = ["1", "2", "3A", "3B"];
```

— and a block screen appears between "who are you" and "how many men", the step
counter goes from 5 to 6, and the value lands in the tracker's Block / Area
column. Nothing else changes anywhere. Leave it empty and that column stays
blank, which the tracker tolerates.

## Firebase

Project `daily-log-d627e` — **shared with Murch**. Reports live in the
`ottawa_reports` collection so the two jobs never mix.

**The Firestore rules must cover this collection or every report is refused.**
Same shape as `reports`: anyone with the link may create, only a signed-in
verified `@unitedservices.work` address may read.

## Data the app produces

One document per report in `ottawa_reports`. The supervisor screen exports a CSV
whose 24 columns are exactly the **Input** tab's columns B:Y, in order, so it
pastes straight in at row 5. `Daily Log` mirrors Input by formula and every
dashboard rolls up from there — nothing else in the workbook is ever written to.

Six columns are blank on purpose: CM, PM, start time, release time, equipment
notes and materials-received notes. Head count is written into **both** the
cover-page and body-text columns, because a single source cannot disagree with
itself; leaving one blank would make the tracker's head-count check read as a
discrepancy rather than a non-question.

Activities Completed is generated from the scopes and quantities. Safety Notes
carries the problem code, hours lost and any typed note.

## Live copy into a Google Sheet

`SHEET_URL` is empty until an Apps Script web app exists for Ottawa — use a
**separate sheet and separate script** from Murch. Source is in `../sheet/Code.gs`;
its token must match `SHEET_TOKEN` here.

Everything else about that path is identical to Murch, including the reason the
POST is `mode: "no-cors"` with `Content-Type: text/plain`, and the separate
`ott_s_sheetbox` retry queue that exists because the sheet POST must not ride on
Firebase's confirmation.

## Project constants baked in

* Clock: `America/Chicago`. Once filed, every date and time is Illinois time
  regardless of where the phone thinks it is.
* Blocks: none yet — add them to `BLOCKS`.
* Baseline quantities: LOTO 14 combiner boxes; MC4, modules removed, modules
  installed and harness wire management 5,478 each. Used only to warn when a
  day's number looks impossible.
* Contract 2026-09-09 to 2026-11-18. 2.80 MW DC.
