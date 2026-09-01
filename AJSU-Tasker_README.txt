AJSU-Tasker | Work Log - Developer README
==========================================
Published by: AJSU | FUTURE STRATEGY
Application: AJSU-Tasker v6
File: v6-files/AJSU-Tasker-v6.html

ARCHITECTURE
------------
Single-file HTML/CSS/JavaScript. No build system, no server, no package manager.
External dependencies: Chart.js v4.4.3 and chartjs-plugin-datalabels v2.2.0
(CDN, pinned, both guarded for offline resilience).
Runs from local filesystem via file:// protocol.
Tested on Chrome and Edge on Windows. Safari not supported.

FILE STRUCTURE
--------------
Everything lives in one file: v6-files/AJSU-Tasker-v6.html
No other files are required to run the application.

EXTERNAL LIBRARIES (pinned — do not upgrade without testing)
-------------------------------------------------------------
Chart.js v4.4.3
  https://cdn.jsdelivr.net/npm/chart.js@4.4.3/dist/chart.umd.min.js

chartjs-plugin-datalabels v2.2.0
  https://cdn.jsdelivr.net/npm/chartjs-plugin-datalabels@2.2.0/dist/chartjs-plugin-datalabels.min.js

SUBRESOURCE INTEGRITY — both tags carry integrity="sha384-..." AND
crossorigin="anonymous". The crossorigin attribute is NOT optional: without it
the CORS check fails and the browser blocks the script even when the hash is
correct, silently disabling every chart.

Bumping either version REQUIRES regenerating the hash:
  curl -sL <url> | openssl dgst -sha384 -binary | openssl base64 -A

A stale hash produces no error dialog — charts simply never render. Test group
36 asserts both attributes are present and that Chart actually loaded under them.

DATA PERSISTENCE
----------------
Primary:   localStorage key "AJSU_Tasker_Record_<PAY_YEAR>"
Secondary: IndexedDB "AJSU_Tasker_DB_<PAY_YEAR>", store "stateStore", key "main"

PAY_YEAR is derived at runtime from the payPeriods array. Changing the year
data changes the storage keys automatically.

On load: IndexedDB and localStorage are both checked; whichever has the newer
  _savedAt timestamp wins (see saveState/loadState) — not IndexedDB unconditionally.
  migrateState() then upgrades the loaded state to the current schemaVersion.
On input: synchronous write to both stores on every input event — no debounce.
On period switch: immediate extractState() then saveState().
On export: extractState() + audit + payYear/schemaVersion stamp + JSON write.
On import: payYear cross-check (confirm if it doesn't match PAY_YEAR) +
  DOMParser sanitize (ALLOWLIST) + validate/coerce + migrateState() + audit +
  state=imp (the SINGLE COMMIT POINT) + saveState() + restore to the tab the user
  was on before clicking Restore.

DEPLOYMENT ENVIRONMENT CONSTRAINTS
-----------------------------------
These DO NOT work in sandboxed iframes (Claude Desktop preview, embedded viewers):
  - window.print()
  - localStorage (throws SecurityError — null-origin)
  - Inline onclick in dynamically injected HTML (CSP blocked)
  - document.createElement() in some null-origin contexts

ALL features work when the file is opened directly in Chrome or Edge.
Do not debug these as app bugs. They are environment constraints.

STATE SCHEMA (current: schemaVersion 2 — migrateState() upgrades older-schema
data on load, so this number will keep changing as the schema evolves without
losing existing users' saved data)
--------------------------------
{
  "schemaVersion": 2,
  "periodId": <integer 1..payPeriods.length>,
  "activeTab": <string>,
  "tasks": {
    "j{julian}y{payYear2digit}_w{isoweek}_r{index}": {
      "activity": "<html string>",
      "category": "Leave/PTO|Meeting|Non-Recurring|Project|Routine",
      "status":   "Complete|Delegated|Follow-up|In-Progress|On Hold|Pending|Re-assigned",
      "priority": "Elevated|High|Standard",
      "worksite": "<string or empty>",
      "notes":    "<html string>"
    },
    "{gp}_no_activity": true   // day marked as no-activity
  },
  "planning": { "<periodId>": { objectives, empNotes, todo, deadlines, supNotes } },
  "history":  { "<periodId>": { periodId, label, dateRange, totalBlocks, complete,
                                completionRate, escalated, byCategory, byPriority,
                                byWorksite, byStatus, savedAt } },
  "memo-name": "", "memo-dept": "", "memo-sup": "",
  "memo-prepared-on": "", "memo-dates-print": ""
}

SPECIAL TASK KEY PREFIXES
--------------------------
blank_{gp}       — UI blank row. NEVER written to state. Skipped by extractState()
                   and aggregateBlocks(). Collision-proof with real task keys.
{gp}_no_activity — Boolean true. Day intentionally empty. Preserved through
                   import/export. Skipped by all metric calculations.

ROW KEY FORMAT
--------------
j{julian}y{payYear2digit}_w{isoweek}_r{ts}_{n}   — every real row, whether
                                    promoted from the day's blank_{gp} placeholder
                                    on first input (see promoteBlankRowIfNeeded) or
                                    created via + Add Row (see addNewRow). Both call
                                    the same newRowId(gp) helper. No other row-id
                                    form exists.

The y{payYear2digit} suffix disambiguates julian day-of-year, which repeats every
calendar year (Jan 5 2026 and Jan 5 2027 are both jd=5) — without it, restoring a
cross-year backup could silently collide keys with current-year data.

Example: j201y26_w30_r1737400000000_4

CRITICAL FUNCTIONS
------------------
promoteBlankRowIfNeeded(t)
  Called from the global 'input' listener, before extractState(). A fresh day
  renders only the blank_{gp} placeholder row. Since blank_ rows are never
  written to state, typing directly into that placeholder must first "promote"
  it: mints a real row id via newRowId(gp), strips the task-row--blank class,
  rewrites the row's child <select> ids/names off the old blank_ id, and
  appends a fresh blank_{gp} placeholder after it. Without this, a brand-new
  day's only visible row silently loses its data on the next period switch.

extractState()
  Updates state.tasks in-place from current DOM rows.
  Skips blank_ prefix rows and no-activity day rows.
  MUST be called before any period switch or import.

capturePeriodSummary(d, pidOverride)
  ALWAYS pass pidOverride explicitly. Never rely on state.periodId during a switch.
  Cross-validates DOM aggregate count against state.tasks for the target period
  using julian day ranges. Falls back to DOM-independent recomputation from
  state.tasks if counts don't match.
  Called with: capturePeriodSummary(null, oldPid) in updatePayPeriod
               capturePeriodSummary(d, pid) in applyStateToUI
               capturePeriodSummary(null, state.periodId) in exportData

aggregateBlocks(pidForWeeks)
  Single DOM traversal for all analytics. Skips blank_ rows and no-activity rows.
  Week determined by ISO week extracted from row ID (not DOM closest() traversal).
  PASS THE PID EXPLICITLY when the on-screen rows belong to a period other than
  state.periodId. extractState() adopts the selector's NEW value, so during a
  switch the global has already advanced; defaulting to it compares the OLD
  period's rows against the NEW period's week-2 ISO week and files every row
  under Week 1. capturePeriodSummary() passes its pid through for this reason.
  Returns: {byStatus, byCategory, byPriority, byWorksite, allRows, total, complete,
            escalated, wsWithData, wsNoData}

generateDays()
  Rebuilds week1-container and week2-container innerHTML.
  After injection, restores ALL select values from state.tasks:
    category, status, priority, AND worksite (worksite was missing in v1 — caused
    all worksite data to be silently overwritten with empty string on every save).
  Resets blank_ row selects to ROW_DEFAULTS (Routine/Pending/Standard) — NOT to
  index 0. Index 0 is an artifact of object key order, not an intended default,
  and for Category it is Leave/PTO.
  Also calls updatePrintIdent() so the print header follows the period switch.

updatePayPeriod()
  Sequence: extractState() → savePlanningForPeriod(oldPid) →
  capturePeriodSummary(null, oldPid) → state.periodId = newPid →
  loadPlanningForPeriod() → requestAnimationFrame(generateDays → buildMemoTable →
  updateViews → saveState)

importData()
  Sequence: reset file input → confirm-if-unsaved → parse → payYear cross-check →
  coerce enums → DOMParser sanitize (strip) → sanitize memo/planning fields →
  coerce history numerics → migrateState(imp) → audit empty-activity tasks →
  restore prior activeTab → state = imp (SINGLE COMMIT POINT) → saveState() →
  applyStateToUI() → recompute history → saveState() → success alert

  ONE COMMIT POINT. Everything before it operates on the detached `imp` object.
  `state = imp` used to sit BEFORE migrateState() and the audit, so any throw
  after it left the rejected backup's rows sitting in the live state while the
  user was told "Restore failed" — and the next keystroke persisted them, merging
  a rejected file into the user's own record. One null row value was enough to
  trigger it. A restore that reports failure must leave the user's data untouched
  in BOTH memory and storage. Do not hoist the assignment back up.

  Both audit loops guard on `typeof !== 'object'`, not only on the _no_activity
  key suffix. A row value that is null or a primitive has no .activity to read;
  reading it throws and aborts the whole restore (or export) over one bad row.
  Same defect class as the _no_activity boolean crash, different value shape.

  migrateState() MUST run before the audit, so an older-format backup is upgraded
  to current-format keys before anything inspects them. Auditing first reads the
  un-migrated keys and drops rows a code update would have preserved.

  migrateState() is also the TASK_KEY_RE security choke point — see below.

  strip() — THE HTML SANITIZER — IS AN ALLOWLIST, NOT A DENYLIST.
  It permits 12 tags (DIV BR P SPAN B I U EM STRONG UL OL LI), removes dangerous
  subtrees outright (SCRIPT STYLE IFRAME OBJECT EMBED LINK META BASE TEMPLATE SVG
  MATH CANVAS AUDIO VIDEO SOURCE TRACK IMG INPUT BUTTON TEXTAREA SELECT FORM
  NOSCRIPT), unwraps everything else, and strips EVERY attribute from what
  survives. It was previously a denylist (remove script/style + on*/javascript:),
  which let through every element nobody had thought of. Three were confirmed
  exploitable from a crafted backup file:
    <iframe srcdoc="<script>...">        executed script in the file:// origin
                                         AND persisted across reloads
    <meta http-equiv="refresh" ...>      navigated the work log away
    <link rel="stylesheet" href="http">  live off-origin request (beacon + a
                                         CSS-selector exfiltration channel)
  The paste enforcer restricts every contenteditable field to text/plain, so the
  only markup that can legitimately appear in a task field is what contenteditable
  itself emits for line breaks. Enumerating what is SAFE is the only filter that
  stays correct against markup that has not been invented yet.

  All three sanitizer passes are scoped to doc.body. doc.querySelectorAll('*')
  also returns <html>, <head> and <body>; unwrapping the document element throws,
  and strip() then falls through to its catch-all "remove every tag" fallback.
  That is safe but it destroys the <div>/<br> boundaries fieldText() rebuilds
  multi-line entries from, so every multi-line activity welds into one line.

  Worksite is enum-checked against VALID_WORKSITES, like the other three selects.
  It was the one select value accepted as a free string.

  (Historical: this sequence used to begin with clearTimeout(_saveTimer), because
  a pending debounced save firing after import silently deleted the just-imported
  data. Saves are synchronous now and _saveTimer no longer exists — but if a
  debounce is ever reintroduced, that guard must come back with it.)

migrateState(state)
  Schema upgrade AND the security choke point for both load paths and import.
  Enforces TASK_KEY_RE — the only permitted state.tasks key shapes:
    j{jd}y{yy}_w{iw}_r{...}   or   {gp}_no_activity
  Sanitizing task VALUES is not sufficient: keys are interpolated into HTML
  attribute context by buildRowHTML() (id="${rId}", data-row-id="${rId}"). A key
  such as   j200y26_w30_r1"><img src=x onerror=...>   was confirmed to give
  persistent, zero-click script execution in the file:// origin that survived
  reload, because the poisoned key was written verbatim to localStorage.
  A malformed key has no legitimate meaning — DROP it, never repair it.

parseRowId(id)
  The only permitted parser for a real row id. Returns {julian, year, isoweek},
  or null for ids that don't match (e.g. blank_ rows) — callers must handle null.
  capturePeriodSummary() and aggregateBlocks() both use it rather than
  hand-rolling separate regexes.

fieldText(elOrHtml)
  The ONLY correct way to read a contenteditable row field as plain text.
  NEVER use .innerText for this. innerText is layout-dependent: inside a hidden
  tab there is no layout, so it degrades to textContent semantics and drops the
  newlines that <div>/<br> boundaries represent — the same row read differently
  depending on which tab the user was standing on. fieldText() derives from the
  markup instead, and accepts either a DOM element or a stored markup string.

currentPeriodId() / DEFAULT_PERIOD_ID
  The opening period is DERIVED from today's date, never hardcoded. It was once
  pinned to 16 in five places, correct only for the fortnight it was typed; the
  app later opened on a closed period, so entries typed into it filed under the
  wrong period. Dates compare as 'YYYY-MM-DD' strings to avoid Date timezone
  drift near midnight. Test fixtures needing a specific period must select it
  explicitly rather than assuming the default.

ROW_DEFAULTS
  Single source of truth for a row's default field values
  ({category:'Routine', status:'Pending', priority:'Standard'}). Read by
  addNewRow(), promoteBlankRowIfNeeded(), clearRow(), and generateDays()'s
  blank-row select reset. Replaced four independently-typed copies with nothing
  keeping them in sync. Applied ONLY to selects the user never touched, tracked
  via a data-touched stamp — applying them unconditionally reverted a deliberate
  Meeting/In-Progress/High choice the instant the activity was typed.
  Note: selectedIndex === 0 is NOT a usable proxy for "untouched", because
  Leave/PTO is itself the first category option.

setSaveStatus(ok)
  Positive save confirmation ("Saved 1:22 PM") in an aria-live region. Runs on
  every keystroke via saveState(), so it must only touch the DOM when the
  minute-resolution label actually changes — otherwise a screen reader reads the
  clock aloud continuously while the user types.

updatePrintIdent()
  Builds the print-only name/department/period header on both week tabs. Called
  from generateDays() AND from the global input listener for memo-name/memo-dept/
  memo-sup, because those fields live on the Memo tab and are typically filled in
  after the worklog rendered — building it once left the printed worklog stamped
  "(name not entered)" all session. Lands via innerHTML, so all three fields are
  escHtml()'d: a restored backup can carry markup into them.

saveState() — banner rule
  NEVER assign banner.textContent. The banner holds an inner
  #unsaved-banner-text span AND the dismiss button; writing textContent on the
  container deletes the button and leaves an undismissable fixed bar over the
  content. saveState() writes through the span.

buildMemoTable(d)
  4-column Tasker Memo: Date | Activity | Status | Notes
  Category and Worksite excluded.
  Week split by ISO week from row ID (not DOM closest).
  Week 1 range: period start +1 (Monday) to +5 (Friday).
  Week 2 range: period start +8 (Monday) to +12 (Friday).

BUTTON BEHAVIOR
---------------
All rows (default, restored, added, blank): Clear + Remove
  Clear  — resets row content to ROW_DEFAULTS, preserves row in DOM.
           Confirms first IF the row has activity OR notes text.
  Remove — deletes row from DOM and state.tasks.
           Confirms first IF the row has activity OR notes text
           ("Remove this row? This cannot be undone.").
           An already-empty row is removed with no prompt — there is nothing
           to lose, and prompting on every empty row trains users to click
           through the confirm that actually matters.
Blank rows: Clear + Remove (Remove dismisses the open slot)

buildRowHTML() always emits BOTH buttons regardless of row origin (isBlank only
toggles the task-row--blank class, nothing button-related). There must be no
unremovable row; the predecessor Excel workbook allowed deleting any row and
this app preserves that guarantee.

ROW GRID INVARIANT
------------------
.task-row's grid track count MUST equal buildRowHTML()'s child count (8).
With 7 tracks, Remove — an irreversible delete — wrapped to an implicit second
grid row at column 1, sitting directly under the NEXT row's Category cell.
Test group 35 asserts tracks === children.

EVENT DELEGATION
----------------
All dynamic row buttons use data-action attributes, not inline onclick.
document.addEventListener('click') handles: clear, remove, add
document.addEventListener('change') handles: no-activity checkbox

ESCALATED STATUSES
------------------
const ESCALATED = new Set(['On Hold','Follow-up','Delegated','Re-assigned'])
Triggers: red BAN in Management View, Items Requiring Review list,
escalated count in capturePeriodSummary.

TASKER MEMO FORMAT
------------------
Title: TASKER MEMO (not Approval Memo)
Header: 2-column (Prepared On / Name / Department | Pay Period / Supervisor)
Activity Tracker: 4 columns (Date | Activity | Status | Notes)
Week grouping: dark WEEK 1 / WEEK 2 headers with date ranges
Status colors: Complete = green, In-Progress = muted gray
Worksite and Category excluded from memo.

COLOR SYSTEM
------------
--color-nav:      #0052CC  Blue  — UI chrome
--color-complete: #007B55  Green — Complete status only
--color-alert:    #D32F2F  Red   — Escalated statuses only

PRINT BEHAVIOR
--------------
@page margin: 0.5in
Any active tab prints. Nav and .no-print elements hidden.
Blank rows (.task-row--blank) hidden in print.
No-activity days: all rows hidden in print.
Print Memo button: switchTab('memo') then window.print() after two nested
  requestAnimationFrame calls (waits for an actual completed paint cycle
  instead of a fixed delay, so it scales with real device load).
window.print() is blocked in sandboxed iframe contexts.

KNOWN LIMITATIONS
-----------------
- Single-user, single-device — data per browser profile per machine
- No undo within session — Remove is permanent
- window.print() blocked in sandboxed iframes
- document.execCommand('insertText') deprecated — monitor browser support
- Chart.js requires internet on first load; cached after that
- Safari not supported — localStorage unreliable for file:// origins

MANUAL TEST CHECKLIST
---------------------
Run automated suite first: node run_test.js (37 groups, 111 assertions)
There are two copies of run_test.js (repo root and v6-files/) — they must be
byte-identical. Check with: diff run_test.js v6-files/run_test.js (must print
nothing). Edit one copy, then copy it over the other.
Playwright must be installed OUTSIDE any Google Drive-synced folder; Drive sync
zeroes every file in node_modules/playwright. See testing.md → Setup.

Then verify manually in Chrome and Edge:

1. IMPORT/EXPORT ROUND-TRIP
   a. Export backup from PP16 → verify JSON has 26 tasks with correct fields
   b. Restore backup → confirm task count alert matches JSON
   c. Switch to Week 1 tab → all rows visible with correct data
   d. Memo tab → Week 1 header shows Mon-Fri dates, Week 2 header shows next week
   e. Switch to PP17 (empty) → PP16 data not visible, PP17 blank rows only
   f. Switch back to PP16 → PP16 data restored correctly
   g. Export again → history shows PP16 only with correct counts, PP17 not stamped

2. WORKSITE PERSISTENCE
   a. Set worksite on 3 rows in PP16 → save → reload page
   b. Worksite values are restored on reload (not reset to empty)
   c. Export → JSON shows correct worksite values
   d. Restore backup → worksites appear in Employee View charts

3. NO ACTIVITY
   a. Check "No activity" on Monday → rows hidden, blank row hidden
   b. Switch period and back → Monday still shows no-activity state
   c. Export → backup has {gp}_no_activity key for that day
   d. Metrics do not count that day

4. ROW OPERATIONS
   a. Every row (default blank, promoted, restored from backup, added) has Clear and Remove
   b. Remove deletes immediately, metrics update without tab switch
   c. Clear resets content, row stays
   d. Typing into a fresh day's blank row promotes it to a real row and it
      survives a period switch / reload (promoteBlankRowIfNeeded)

5. MEMO
   a. TASKER MEMO heading
   b. Week 1 header: JULY 20 — JULY 24 (Monday, not Sunday July 19)
   c. Week 2 header: JULY 27 — JULY 31
   d. Both weeks visible on screen (scroll if needed) and in print
   e. Worksite and Category not in Activity Tracker table

6. HISTORY / TRENDS
   a. Switch PP16 → PP17 → PP16 → export
   b. PP15/PP17 history NOT stamped with PP16 data
   c. PP17 with no tasks shows totalBlocks: 0 in history (not 26)
   d. Trends tab shows only periods with totalBlocks > 0

CONSOLE COMMANDS FOR TESTING
------------------------------
View state:
  JSON.parse(localStorage.getItem('AJSU_Tasker_Record_2026'))

View history:
  JSON.parse(localStorage.getItem('AJSU_Tasker_Record_2026'))?.history

Clear localStorage:
  localStorage.removeItem('AJSU_Tasker_Record_2026')

Clear IndexedDB:
  indexedDB.deleteDatabase('AJSU_Tasker_DB_2026')

Full reset:
  localStorage.removeItem('AJSU_Tasker_Record_2026');
  indexedDB.deleteDatabase('AJSU_Tasker_DB_2026');
  // Then reload
