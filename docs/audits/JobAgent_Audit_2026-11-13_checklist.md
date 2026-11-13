# JobAgent audit (2026-11-13): implementation checklist

Every item in `JobAgent_Audit.md` and what happened to it. Status is one of
Fixed, Implemented, Already resolved, Deferred, or Rejected. Each code change
has a regression test, named in the last column.

## Confirmed bugs

| ID | Status | What was done | Test |
| --- | --- | --- | --- |
| BUG-01 | Fixed | Every screening-answer write keeps `profileOwnedAnswerIds` and `profileId`. Each answer's own `profileId` stamp survives a whole-list save from the editor, which was a second leak the audit missed. `effectiveAnswers` withholds any answer stamped with another profile. `loadConfig` restores the guard in memory if an older build stripped it. | multiProfileAnswerIsolation |
| BUG-02 | Fixed | `question_queue.json` and `semantic-rerank.json` resolve through `workspacePaths(root).dataDir`. The original workspace keeps `data/`, so no migration is needed. | workspaceIsolation |
| BUG-03 | Fixed | New `data/dashboard.lock` (`{pid, port}`): `app` reuses the live holder's port, and `dashboard` refuses to start a second server. The server also stats the activation and config files on each API request and rebinds or reloads when another process changed them (never mid-run). | dashboardInstanceSync |
| BUG-04 | Fixed | Editing the active profile during a run is refused with 409 before anything is written (`notSaved: true`). | dashboardInstanceSync |
| BUG-05 | Fixed | `restore-version` checks workspace containment for the folder and every target, restores or rebuilds `letterDocx`, sends an approved packet back to review, records an event, invalidates caches and refreshes the tracker. | documentVersionsRestoreApproval |
| BUG-06 | Fixed | The popup opens the posting with `chrome.tabs.create` (http/https only). The content script treats `aria-required` comboboxes, radio groups and custom widgets as required. | extensionAriaRequired, extensionE2E |
| BUG-07 | Fixed | `preparePacketsForJobs` returns `cancelled` separately. Items are labelled prepared, failed, cancelled or skipped, and cancelled is styled neutral, not red. | prepareBatchCancel |
| BUG-08 | Fixed | `hideApplied` on the Jobs list, plus the `all` view predicate, leave out submitted, submit-unverified and skipped jobs. Applications and "Closed jobs" still show them. | jobsHideApplied |
| BUG-09 | Fixed | Unsubmitted work on postings over 21 days old sinks into an "Aging" Today group with an inline Archive. Filled forms and unverified submits are never demoted. | todayAging |
| BUG-10 | Fixed | Table-wide groupings are memoized on the data key (real store: 153 ms cold, 13 ms warm). "Active" is the Jobs list's own count (2,982 = 2,982 on a copy of the real store). | dataQualityActive |
| BUG-11 | Fixed | View switches sync the rail to the new list (keep the job if listed, else open the first item on wide screens, else clear). Profile switches clear the `#job=` hash and the selection. | dashboardLayoutA11y |
| BUG-12 | Fixed | One roving tab stop for the board (300 Tab stops became 3), with ↑↓/j/k/Home/End. File inputs are named, and the pager is a `<nav aria-label="Pagination">`. | dashboardLayoutA11y, workbenchUi |
| BUG-13 | Fixed | `displayCompany()` at ingestion, on board rows and in letters. The job ID is computed from the raw name, so no existing row is re-keyed. | companyDisplayName |

## Architectural risks

| ID | Status | What was done | Test |
| --- | --- | --- | --- |
| RISK-01 | Fixed | Idle workspace DB handles are closed after a switch, when nothing is running. The boot handle is never closed. | dashboardInstanceSync |
| RISK-02 | Fixed | The extension `/result` checks and write run in one `BEGIN IMMEDIATE` transaction, with the checks repeated under the lock. | extensionCompanion |
| RISK-03 | Fixed | A tracker write is bound to the `(db, root)` it was scheduled for, and a pending write is flushed before a switch takes over. | dashboardServer |
| RISK-04 | Fixed | One shared Chromium per batch, a fresh context per PDF, closed after 15 s idle and on CLI or server exit. | chromiumPdf |

## Innovation backlog

| # | Feature | Status | Notes |
| --- | --- | --- | --- |
| 1 | ATS auth-state and tenant batching | Implemented | Sign-ins already persist (one browser profile per person). Added grouping by portal host and "Retry all N" on Today and Applications, using the ordinary batch Auto-fill (confirms first, stops before Submit). Test: autofillReadiness |
| 2 | IC vs. people-manager track | Implemented | Penalty-only `trackFit` (−20, or −12 the other way). `SCORER_VERSION` 3. The Explore Score Lab mirrors it. Test: managementTrackScoring |
| 3 | Employer sponsorship memory | Implemented (corpus part) | Per-employer counts from the store, shown as context only, with no score change. The offline DOL LCA lookup is deferred: it needs a large external dataset. Test: employerHistory |
| 4 | Command palette and keyboard triage | Implemented | Ctrl/⌘K palette (combobox pattern, no animation) and s/e/r/p on the focused row. Test: dashboardLayoutA11y |
| 5 | Pre-flight autofill readiness | Implemented | Access type (no account vs. employer sign-in) and that job's open questions, in the detail rail. Test: autofillReadiness |
| 6 | Resume-to-JD evidence map | Implemented (one-way) | Each asked-for skill is quoted against the master resume. Click-to-highlight in both directions is deferred. Test: evidenceMap |
| 7 | Ghost / repost detector | Implemented (repost part) | Same title at the same employer 3+ times over 30+ days. Staffing-agency and vague-JD heuristics are deferred. Test: employerHistory |
| 8 | Preference calibration | Implemented | Proposals from decision reasons, using existing undoable actions only. Test: preferenceCalibration |
| 9 | Zero-chrome split view | Partially implemented | The first row moved from 369 to 264 px at 1280×800; a floating bulk bar; the Jobs heading is hidden visually. A full layout rewrite was rejected (see below). |
| 10 | Smooth freshness decay | Implemented | Apply-window bands within a tier. Test: freshnessBands |
| 11 | Question clustering | Deferred | Needs answer-flow UI changes in the profile drawer. |
| 12 | Cross-source requisition linking | Deferred | Requisition-aware dedupe already exists (P1-4). URL upgrading needs more design. |
| 13 | Pooled packet compiler | Partially implemented | Chromium is pooled (RISK-04). A persistent Python tailor worker is deferred: it changes the process protocol. |
| 14 | Inline AI micro-actions | Deferred | Optional AI; the deterministic clause strip (#15) covers the motivating example. |
| 15 | Hidden-clause callouts | Implemented | Quote-backed and deterministic. Test: hiddenClauseExtractor |
| 16 | Funnel telemetry | Already resolved, plus addition | Source, ATS, score-band and tier breakdowns existed. Added apply speed. Test: conversionBreakdown |
| 17 | Follow-up cadence | Already resolved | Follow-ups, due reminders on Today and "Done" existed. Drafting follow-up notes is deferred. |
| 18 | Slug canonicalizer | Implemented | Same as BUG-13. |

## UI/UX findings

| Finding | Status |
| --- | --- |
| Vertical chrome tax | Fixed. 369 → 308 px to the first row at 1280×800 (264 px before the always-present 44 px toolbar) (the audit's 512 px assumed the filter panel was open, which is not the default). |
| Bulk bar covers `<thead>` | Fixed. A persistent 44 px table toolbar docked above the headers (both sticky); the apply pipeline is one Prepare › Approve › Auto-fill stepper. A floating version was tried first and dropped: it blended into the rows and could hide the focused row. |
| Empty right rail on Today | Fixed. The first card opens automatically on wide layouts. |
| 11 px micro-type | Fixed. `--fs-xs` is now 12 px. |
| No inline triage on Today | Fixed. "Not for me" and "Archive" with Undo. |
| Applications bands overflow | Fixed. Each column scrolls on its own above 860 px. |
| Repeated "Needs sign-in" cards | Fixed. Grouped by portal (backlog #1). |
| Raw internal keys in the status bar, duplicated filter chips across three surfaces | Deferred. These need an information-architecture pass across the header. |
| Tabbed inspector (four sub-tabs), left nav rail | Rejected for now. It is a broad rewrite of a working, heavily tested layout. Clause tags, readiness, evidence and history were added to the existing rail instead. |
