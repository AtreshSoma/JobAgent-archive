# JobAgent audit, 2026-11-11

Audit only. No source file was changed. Every run used a throwaway workspace (`JOB_AGENT_ROOT`), local HTML fixtures and fictional test personas. No application was submitted, no employer or job site was contacted, and the live app on port 4646 was never contacted. The real database file was only ever opened read-only. Its hash was identical before and after the baseline suites; it changed once later (file time 21:17 local) while the owner's app, running since 20:49, held it open. Read-only checks afterwards found no job, event, approval or run row newer than 20:51 local (before the audit began), no audit-seeded ids, unchanged schema version 4 and `quick_check` ok, so the change is attributed to the running app (most likely a WAL checkpoint). That attribution is an inference, not a proof.

Labels used throughout:

- **Confirmed**: reproduced by running the product code in this audit.
- **Static**: shown by the code, not executed.
- **Unverified**: plausible, not proven either way.

Scratch material (repro fixtures, harness, benchmark, raw results) is in the session scratchpad and is not part of the repository. Appendix A repeats what is needed to reproduce each confirmed finding without it.

---

## 1. Executive summary

**Verdict.** The safety gates around the final submit hold in the shipped configuration, and the test baseline is green. The autofill layer underneath them is not yet trustworthy: on local fixtures it wrote wrong answers into forms, answered sensitive questions it should have left blank, clicked a control belonging to a different question, clicked a final "Apply now" button outside the submit gate, and reported unfinished forms as ready. Ranking is stable and deterministic but does not model the requirements that decide eligibility in licensed professions, and it does not separate required from preferred skills. Personal data is still reachable in a public repository.

**Top risks**

| # | Risk | Evidence |
|---|---|---|
| 1 | Personal data in public git history: the archive repository is public and 10 commits on its `main` touch personal-data paths (values not inspected) | Confirmed, SEC-01 |
| 2 | Wrong or untruthful answers reach an employer form: own name written into "Company name", "Reference name", "Middle name"; residence country written into "Country of citizenship"; "Yes" to "Have you ever failed a background check"; "Yes" to an accommodation question | Confirmed, AF-02, AF-03 |
| 3 | A final "Apply now" is clicked without the submit gate when the scanner cannot see the form's inputs | Confirmed, AF-01 |
| 4 | A typeahead with no matching option clicks an unrelated control elsewhere on the page (in the repro, the "Yes" of a sponsorship question) | Confirmed, AF-05 |
| 5 | Partial work reported as done: `ready_for_submit` with required questions blank, with unanswered radio groups, and after seven no-op clicks on "Next" | Confirmed, AF-06, AF-07, AF-08 |
| 6 | Auto-submit can record `submitted` for a click that did nothing. Not reachable from the dashboard today because the live-verified list is empty, but live the moment one ATS is listed | Confirmed, AF-04 |
| 7 | Approval is not bound to content: answers edited after approval are what gets sent | Confirmed at runner level, AF-10 |
| 8 | In this checkout the dashboard token, the SSRF guard, backups and encryption exist only as uncommitted files (163 dirty paths over HEAD `d536857`) | Confirmed, SEC-02 |

**The 5 highest-value actions**

1. Make the archive repository private today and ask GitHub Support to purge cached views (SEC-01). Nothing else on this list matters as much.
2. Tighten the two matchers that decide what is typed: the label classifier (AF-02) and the screening matcher (AF-03). Both are pure functions with table-driven tests, so this is a small change with a large effect on correctness.
3. Close the three paths that act outside their intended scope: "Apply" as an advance control (AF-01), the page-wide option search (AF-05), and the confirmation check without a pre-click baseline (AF-04).
4. Make "ready" mean ready: one required-field definition used by both assist and auto-submit, question labels for groups, and a progress check after every advance click (AF-06, AF-07, AF-08).
5. Commit the working tree of this checkout, or retire the checkout in favour of the public clone, before any further work (SEC-02).

---

## 2. Baseline

| Item | Value |
|---|---|
| Commit | `d536857` on `main` (merge of PR #3), origin = `AtreshSoma/JobAgent-archive` |
| Working tree | 163 changed or untracked paths (73 tracked files modified, +3968/-1718 lines, plus about 90 untracked). The audit covers the working tree as it stands |
| Runtime | Node v25.9.0 (CI treats 25.x as experimental; supported lines are 22.x and 24.x), Python 3.12.4, macOS (darwin) |
| Sibling public clone | `../JobAgent-public` at `62b98b7`, 26 dirty paths. Not audited; used only to establish what is committed there |

| Suite | Command | Result |
|---|---|---|
| Repo health | `JOB_AGENT_ROOT=<temp> npm run verify` | PASS, 6 of 6 checks, "0 job(s) in store" |
| Lint | `npm run lint` | exit 0, 0 errors, 21 warnings. 8 of the 21 come from the untracked nested clone `JobAgent/` because eslint does not ignore it. Python lint skipped: `ruff` is not installed locally |
| Typecheck | `npm run typecheck` | exit 0, no output |
| Node tests | `npm test` | 216 files passed, 2815 tests passed, 3 skipped, 62.75 s |
| Python tests | `python3 -m pytest` | 95 passed, 4.54 s |

Each suite was run once. No retries were needed.

**Could not run here**

- Windows and Linux: not available locally. Public CI at `62b98b7` is green on all 10 jobs including four Windows cells; the previous run on `c4246a6` failed on `node 22.x / windows-latest` only.
- `ruff`: not installed. CI installs it.
- `npm run test:coverage`: not run (thresholds are enforced on one CI cell).
- Firefox: cannot launch in this environment. Browser work used Chromium.
- Any live ATS: out of scope by rule.

---

## 3. Bug register

Severity: Critical, High, Medium, Low. Each Evidence cell gives the label and the confidence. R-numbers refer to Appendix A.

### 3.1 Autofill and application automation

| ID | Sev | Component | Problem (expected vs actual) | Evidence | Repro | Root cause | Fix | Test |
|---|---|---|---|---|---|---|---|---|
| AF-01 | High | navigator, runner | Expected: a final submit is only ever clicked by `tryAutoSubmit`. Actual: on a quick-apply form whose inputs have no `id` or `name`, the run clicked "Apply now", the page showed "Thank you for applying", and the run reported `assist_blocked: no application form found` | Confirmed, high. `applyRunner.js:735-737`, `navigator.js:139-142`, `domScanner.js:194-195` | R7 | `allowApplyStart` is true when the scanner returns no fields and nothing was filled. Text inputs with no derivable selector are dropped by the scanner (`if (!sel) continue`), so a real form looks like a landing page. Shadow DOM and non-allowlisted iframes have the same effect (Static) | Decide "landing page" from a raw count of visible `input, textarea, select` (including shadow roots), not from the filtered scan. Give id-less controls a per-scan handle as `navigator.js` already does for buttons. Treat an Apply click that lands on confirmation text as `submit_unverified` | Fixture: id-less inputs plus "Apply now"; assert zero clicks and `assist_blocked` with a reason naming unscannable fields |
| AF-02 | High | formPrefill | Expected: profile values only in the field that asks for them; citizenship never inferred. Actual: full name written into "Middle Name", "Current company name", "Reference name"; own email into "Reference email"; own phone into "Emergency contact phone"; residence country selected for "Country of citizenship" | Confirmed, high. `formPrefill.js:113-131` (name), `:153-161` (phrase match), 37-label probe | R4 | `classifyNameField` returns `fullName` for any label of 8 words or fewer containing "name". Remaining aliases match any label containing the alias word, with one disqualifier list (phone only) | Require the label head to be the alias (label equals alias after removing "your", "legal", "*", "(required)"), add disqualifying nouns (company, employer, school, university, reference, emergency, manager, recruiter, referred, middle, maiden, previous, degree, pronunciation, signature), and route citizenship, birth and nationality to the sensitive path | Table-driven `classifyLabel` test from the 37 probe labels; end-to-end fixture asserting those inputs stay empty |
| AF-03 | High | screeningQA, shipped example config | Expected: unknown or differently-scoped questions stay blank. Actual: "Have you ever failed a background check or drug screen?" answered Yes; "Will you require any accommodation..." answered Yes from the sponsorship entry; "Will you require relocation assistance?" filled with the sponsorship sentence; "How many hours per month can you travel?" filled with the monthly salary | Confirmed, high. `screeningQA.js:477-497`, patterns in `config/screening_answers.example.json` (`background check`, `drug screen`, `will you require`) | 22-question probe, R4 | A pattern scores 1.0 when it appears anywhere in the question as a phrase. Polarity and subject are checked only for sponsorship and years-of-experience | Add a negative-history guard (failed, denied, convicted, terminated, refused, revoked) and a sensitive-topic deny list (accommodation, disability, criminal, age, religion, pregnancy) that always return `ask_user` unless an entry is tagged for that topic. Require the pattern to cover the question's main clause. Remove `will you require`, `per month`, `how many years` from the shipped patterns | Corpus test: each stored answer against 5 near-miss questions must return `ask_user` |
| AF-04 | High, gated | runner verification | Expected: `submitted` only on new evidence. Actual: Submit handler did nothing, page showed its own error, run returned `submitted: true`, phase `verified`, on the sentence "Once your application is submitted, you will not be able to edit it" that was on the page before the click | Confirmed, high. `applyRunner.js:1176-1188`, `:1337-1342` | R1 | `SUBMIT_CONFIRMATION_RE` is tested on post-click text with no pre-click baseline. The hand-off watcher has a baseline (`baselineHit`), the auto-submit path does not. A confirmation phrase alone is sufficient | Snapshot text, URL and field set before the click. Count a phrase only if it is new and either the form is gone or the URL changed. Run `detectValidationErrors` after the click. Reuse `POST_SUBMIT_PAGES` | Fixture with the phrase pre-rendered and a no-op submit must end `submit_unverified` |
| AF-05 | High | fieldFiller | Expected: a typeahead with no match is left for the human. Actual: "Location" typed, no listbox opened, the filler clicked the first element on the page whose class contains "option", which was the "Yes" button of "Do you now or in the future require visa sponsorship?". The run reported Location as filled and `ready_for_submit` | Confirmed on a synthetic fixture, high. `fieldFiller.js:75`, `:100-101`, `:111-136` | R5 | With no `aria-controls` the option search is page-wide (`[class*="option"]`), and short non-sensitive values may commit index 0 without a text match. The read-back checks the typeahead, not what was clicked | Search only inside the popup owned by the input (aria-controls, aria-owns, or nodes added after typing). Remove the top-option fallback except for an exact-prefix match. Verify the read-back equals the intended value | Fixture R5 must record zero clicks outside the field and report the field as needing the user |
| AF-06 | Medium | domScanner | Expected: a radio or checkbox group is identified by its question. Actual: groups in `fieldset > legend` are labelled by their first option ("Yes", "Day"), never answered, and queued for the user under those labels. Run still `ready_for_submit` | Confirmed, high. `domScanner.js:38-57`, `:173` | R2 | `labelFor` never reads `legend`, `role=radiogroup` labelling or surrounding text; the group label is the first radio's own label | Resolve group label from legend, then group `aria-labelledby` / `aria-label`, then nearest heading text. Group checkboxes sharing a fieldset | Scan test on R2 expects the legends as labels |
| AF-07 | Medium | domScanner, runner | Expected: a required unknown question blocks the run. Actual: fields marked `aria-required="true"` and a trailing asterisk were treated as optional; run ended `ready_for_submit` with both required questions blank | Confirmed, high. `domScanner.js:159`, `:173`, `:222` versus `:182-183` | R3 | `required` is the native attribute only, except for checkboxes. The stricter live check (`unfilledRequiredControls`) runs only inside `tryAutoSubmit` | One `isRequired` helper (native, aria, asterisk, "(required)"). Call `unfilledRequiredControls` before declaring `ready_for_submit` and list what is blank | R3 must end `assist_blocked` naming both questions |
| AF-08 | Medium | runner | Expected: a "Next" that does not advance stops the run. Actual: 7 clicks on a no-op "Next", the same field refilled 8 times, then `ready_for_submit` with `advancedSteps: 7` while the page still said "step 1 of 4" | Confirmed, high. `applyRunner.js:791`, `:838-902` | R6 | Success of an advance is inferred from the absence of recognised error markup. The step budget ending is reported as ready even when an advance control is still present | Fingerprint the step (URL, heading, field labels) before and after; unchanged means `assist_blocked: could not advance`. At the step budget report "more pages remain" | R6 must end blocked after one click |
| AF-09 | Medium | fieldResolver, fieldFiller | Expected: a committed typeahead counts as filled. Actual: after a correct commit on a react-select style control (input cleared, value shown beside it) the run blocked with "was not accepted: value did not remain set". Separately, a sentence answer typed into a yes/no typeahead matches nothing and blocks | Confirmed on an imitation of react-select, medium (real Greenhouse not exercised). `fieldResolver.js:160-167`, `fieldFiller.js:47`, `:54-66`, `:331` | R9b, R9 | Verification reads `input.value` for every action except check and radio. Combobox options are unknown at plan time, so `mapToChoice` is skipped and the whole sentence is typed | Verify comboboxes through `readBackChoice`. Open the list, read its options, map with `mapToChoice`, then pick | R9b must end `ready_for_submit` with the choice recorded |
| AF-10 | Medium | approval, runner | Expected: what was approved is what is sent. Actual: packet approved with a salary answer of about 150k; the stored answer was then edited to about 90k; auto-submit sent 90k and reported `submitted` with nothing withheld | Confirmed at runner level, high; dashboard path Static. `applyRunner.js:323`, `:563`, `:345`; `approval_queue` has no content hash | R10 | Dynamic fields are filled from live config, not from the packet. The only comparison with the packet covers labels matching the sensitive regex. `restore-version`, link replace and answer edits do not reset approval (`jobRoutes.js:170-180`) | Store a digest at approval (resume and letter sha256, answers snapshot, profile fields used). At fill time compare and stop with "changed since you approved" or fill from the snapshot | R10 must end blocked with the changed answer named |
| AF-11 | Medium | data, applicationState | Expected: "No, it didn't go through" lets the user retry. Actual: the database row is reset but the checkpoint stays `user_submitted`, so the retry returns `submit_unverified` without opening the browser, every time | Confirmed at runner level, high. `data.js:1952-1977`, `applicationState.js:299-301`, `applyRunner.js:182-190` | R11 | Two stores of truth; the retraction updates one. `freezeVersions` is also never undone (`jobRoutes.js:151`) | Give the checkpoint a human-only retraction transition, call it from `retractSubmissionReport`, unfreeze versions | Route test: mark submitted, retract, assist; expect the form to open |
| AF-12 | Medium | commands.recordAssistResult | Expected: every status change validated and audited. Actual: assist outcomes are written with `db.upsertJob`, not `setJobStatus`; no `status_changed` event; write errors are swallowed unless a submission was claimed; nothing is written when a run starts, so a crash leaves no trace on the job row | Static, high. `commands.js:709-780` | Read the function | The state machine is enforced only in `Database.setJobStatus`; this path predates it | Route through `setJobStatus` with the missing transitions added; write `assist_started` at run start; surface write failures in the run result | Unit: illegal transition rejected; event row present |
| AF-13 | Medium | server batch | Expected: a packet the user rejected is not filled. Actual: "Auto-fill" approves any non-approved packet, rejected included, then fills. Single-job assist ignores an archived or rejected job decision | Static, high. `server.js:1701-1705`, `:1454-1475` | Read the functions | "The click is the approval" is applied to every prior state | Skip rejected packets with a reason; check `user_decision` in `startAssist` | Batch test with one rejected packet expects it skipped |
| AF-14 | Low | atsDetector, platforms | `clever.com` is detected as Lever (substring host match). Oracle, Rippling, Workable and `myworkdaysite.com` are `generic`. Per-ATS selector maps never run in production because nothing populates `job.formFields` (`pipeline/index.js:929`) | Static plus executed call, high | `detectAts({url:'https://clever.com/jobs/1'})` | `host.includes(sig)`; descriptors bind only a static plan that is always empty | Match on host suffix; either delete the descriptors or feed them | Detector table test |
| AF-15 | Low | scanners, guards | Fail-open on errors: `unfilledRequiredControls` returning `null` passes the auto-submit gate (`applyRunner.js:354-355`); an unreadable page gives the guard empty HTML, which is not blocked (`guards.js`, `browser.js:142`) | Static, medium | n/a | Errors are mapped to "nothing found" | Treat inspection failure as a blocker on the submit path | Unit with a throwing `evaluate` |
| AF-16 | Low | server watcher | The 45-minute hand-off watcher writes through a `db` variable that a profile switch rebinds, so a late report can insert a stub row in another person's store | Static, not re-run. `server.js:1510`, `:244` | n/a | Closure over a mutable binding | Capture the handle and job owner at run start | Unit with a rebound handle |

### 3.2 Data integrity and workflow state

| ID | Sev | Component | Problem | Evidence | Root cause | Fix | Test |
|---|---|---|---|---|---|---|---|
| DI-01 | Medium | approvalQueue | Approve has no state check and writes no audit event; re-approve overwrites the only timestamp; reject also writes `approved_at`. Approving after rejection or submission returns 200 | Static, high. `approvalQueue.js:105-124` | Approval is one row with no history | Write an event per approve and reject; refuse approve on submitted jobs; add `rejected_at` | Route tests |
| DI-02 | Medium | prepare | Re-prepare is refused only when `applied = 1`. A `submit_unverified` or `ready_for_submit` job can be re-prepared, which resets it to `prepared` and erases the marker the duplicate guard reads | Static, high. `commands.js:513-518` | Guard checks one column | Refuse for `submit_unverified`; ask for confirmation for `ready_for_submit` | Unit |
| DI-03 | Medium | data | A verified checkpoint with a failed database write leaves `status = submitted, applied = 0`; outcomes and follow-ups then return 409 and the UI offers no repair | Static, medium (path traced, not run). `applyRunner.js:184-186`, `commands.js:752`, `data.js:1987` | Two stores, no reconciliation | On open, reconcile checkpoint and row; let "Mark submitted" set `applied` when status is already submitted | Unit |
| DI-04 | Low | selection.js | Bulk undo restores from in-memory rows (`?? 'clear'`), so a row not loaded loses a prior "shortlisted". The server's exact undo (`/undo-decision`) has no front-end caller | Static, high. `selection.js:193`; no match for `undo-decision` in `public/assets/js` | Undo built client-side | Call the server undo per job | UI test |
| DI-05 | Low | data | `applyDecision` is two statements without a transaction; outcome writes read, validate and insert without one | Static (sweep, spot-checked) | n/a | Wrap in `transaction()` | Unit with a failing second statement |
| DI-06 | Low | mcp | The MCP server opens its database once at start; after a profile switch in the dashboard it writes notes and outcomes to the previous workspace | Unverified (sweep, not re-checked) | n/a | Resolve the workspace per call | Unit |

Not found, confirmed by search: an un-approve action, a per-job delete, a "ghosted" outcome, a packet content hash, any write of `assist_started` before a run.

### 3.3 Ranking

Measured by the benchmark in section 5.

| ID | Sev | Problem | Evidence | Root cause | Fix | Test |
|---|---|---|---|---|---|---|
| RK-01 | High | Licence, certification and degree floors are not modelled. A Nurse Practitioner post (MSN and NP licence required) scored 88 Priority for an RN. A PhD-required post scored 90 Priority for an engineer whose profile lists no education (verdict `unknown`, no cap) | Confirmed, high. n5, s8; degree extraction 10 of 14 (misses "BSN required", "MSN required", "JD or LLM", "Degree in Accounting") | `eligibility.js` knows years, generic degrees and clearance. An unmet floor costs at most 8 raw points; an unknown one costs nothing | Extract required credentials from the requirements section, compare with `certifications` and `education`, and cap the tier at Queue when a required credential is unmet or unknown | Benchmark rows n5, s8 must not reach Apply |
| RK-02 | High | Required and preferred skills score the same. NICU nurse post (required specialty not held, candidate skills only under "Preferred") 87 Priority; Senior Tax Accountant 90 Priority; Frontend Engineer with the candidate's stack under "Nice to have" 72 Queue | Confirmed, high. n12, a6, s5 | `keywordCategory` matches the whole document. The preferred-section parser in `eligibility.js` is used for years and degree only | Reuse `clausesOf` to weight matches: required 1.0, preferred about 0.3; report "required, not evidenced" separately | Paired-posting test: same skills moved between sections must change the score |
| RK-03 | Medium | Seniority is misread for common titles: 14 of 26 correct against my labels. "Staff Accountant" and "Staff Nurse" read as lead; "Account / Case / Project Manager" as lead; "II / III / IV" and "Charge Nurse" unread; "Data Entry Clerk" read as entry level. Effect: an entry-level Staff Accountant post scored 90 Priority for a senior accountant as "seniority +1 (step up)" | Confirmed, high (labels are my judgment; 3 of the 12 misses are debatable) | One alias table for all professions (`scorer.js:58-92`) | Make "staff" and "manager" depend on the role family; read numerals; never read "entry" inside "data entry" | The 26-title table as a unit test |
| RK-04 | Medium | Missing description is scored as a weak match instead of "not judged". A perfect title with an empty description scored 50 to 53, Near miss, for all three personas (40 points below the same post with its text) | Confirmed, high. n10, a9, s10 | `needs_description` is assigned only when the tier is Discard and `summaryOnly` is set (`pipeline/index.js:477-502`) | Use `needs_description` whenever the description is under a minimum length and the title is not rejected | Benchmark rows must show "needs description" |
| RK-05 | Medium | The pipeline's queue order puts "posted in the last 24 h" and sponsorship rank ahead of score. In a constructed scenario (two weak posts re-dated to 3 hours old) NDCG@5 fell from 0.565 to 0.345, 0.778 to 0.491 and 0.682 to 0.472. Sponsorship rank is applied even for candidates who need none | Confirmed, medium (scenario is constructed). `jobRanking.js:46-66`, used at `pipeline/index.js:621-624` | Lexicographic sort | Blend freshness into the score or use it only as a tie-break; skip the sponsorship key when the candidate needs none | Comparator unit test |
| RK-06 | Medium | Dedupe: 5 of 10 constructed cases match the expected decision. Distinct requisitions still merge when descriptions are short, when sources differ, or when they differ only by shift. "Sr." vs "Senior" and "Corp" vs "Corporation" do not merge | Confirmed, high. Section 5.6 | Split requires same source, two ids, both descriptions of 400+ characters and similarity under 0.85 | Normalise title and company abbreviations in the key; treat requisition id plus location or shift as distinguishing; surface merged siblings | The 10 cases as a unit test |
| RK-07 | Low | Years extraction 26 of 28: misses "5+ years of experience (8+ preferred)" and "Experience (years): 5" | Confirmed, high | The preferred cue marks the whole clause; no key-value form | Split on parentheses; add the key-value pattern | Sentence table |

### 3.4 Security and operations

| ID | Sev | Component | Problem | Evidence | Fix | Test |
|---|---|---|---|---|---|---|
| SEC-01 | Critical | Repositories | `AtreshSoma/JobAgent-archive` is public. The repository's own check fails on its history with 6 personal-data paths (candidate profile, screening answers, cover-letter settings, master resume, two database sidecar files); 10 commits on `origin/main` touch those paths. The clean public repository and its fork are not affected; the friend's old fork now returns 404 | Confirmed, high. `gh api` visibility; `node scripts/checkPersonalData.mjs` (paths only, no values read) | Make the archive private; request a purge of cached commits; treat the exposed details as disclosed | CI check already exists in the working tree |
| SEC-02 | High (operational) | This checkout | The dashboard token, connect-time SSRF guard, backups, bundle encryption, secret store, MCP and inbox are untracked or uncommitted here. `git show HEAD:src/dashboard/server.js` contains no token code. A `git stash -u`, `git clean` or partial commit loses or breaks them | Confirmed, high | Commit, or retire this checkout in favour of the public clone | n/a |
| SEC-03 | Medium | `cli/app.js` | The launcher sends the dashboard token as a header to whatever answers on the port with the right JSON shape, then opens `/?token=...` on it. It also prints the unlock URL and passes it in argv | Static, high. `app.js:94-100`, `:269-273` | Probe without the token; identify the server by a nonce file or by pid | Unit with a fake listener |
| SEC-04 | Low | `apiToken.js` | The session cookie is sent to every port on loopback, lasts 30 days, and the token never rotates | Static, high. `apiToken.js:106` | Rotate per start; shorten lifetime | Unit |
| SEC-05 | Medium | `db.js` | A database with a newer schema than the code opens without error or backup (`if (current >= step.version) continue`) | Static, high. `db.js:320-321`, `:543-546` | Refuse to open a newer schema | Unit |
| SEC-06 | Medium | export, backup | Exports are plaintext unless a passphrase is supplied; backups are always plaintext; staging uses the OS temp directory | Static, high for the default (`exportData.mjs:24-32`), sweep for staging | Prompt for a passphrase by default; stage inside `data/` | Unit |
| SEC-07 | Medium | docs vs LLM | `SECURITY.md:34` and `.env.example` say the model receives only the public description and title. With `LLM_ASSIST=on` and `INBOX=imap`, up to 6000 characters of an email body are sent (`inbox/classify.js:85`); rewrite, letter and rerank features send resume bullets | Static, high | Correct the documents; give inbox classification its own switch | Doc test |
| SEC-08 | Low | MCP | `get_job` returns absolute file paths whose names contain the candidate's name, regardless of `MCP_SHARE_CONTACT`; `add_note` replaces the note | Static, high for paths (`mcp/server.js:64-65`) | Whitelist fields; append notes | Unit |
| SEC-09 | Low | automation | A stored job URL is passed to `page.goto` with no scheme or host check in `src/automation` | Static, high (search found none) | Reuse `assertPublicHttpUrl` before navigation | Unit |
| SEC-10 | Low | Windows | No `py` launcher fallback (`pythonEnv.js:86-88`); Windows reserved names not handled in output folders; `start-dashboard.cmd` drops extra flags | Static (first item verified, others sweep) | Add `py -3`; guard reserved names | Unit on `win32` |
| SEC-11 | Low | lint | eslint scans the untracked nested clone `JobAgent/`, which produced 8 of 21 warnings | Confirmed | Add to eslint ignores, or delete the clone | n/a |

Checked and sound (sweep, key points re-read): loopback-only bind; timing-safe token compare; Host and Origin checks on every route; no `innerHTML` in the dashboard and a strict CSP; SSRF guard pins the resolved address per hop for user-supplied URLs; CSV formula escaping; resume upload size, type and zip-bomb limits; migrations run in a transaction after a backup; the extension has no content scripts and never clicks submit.

---

## 4. Autofill deep dive

### 4.1 The path, as built

| Stage | Where | What I found |
|---|---|---|
| Profile, answers | `config/*.json`, `screeningQA.effectiveAnswers` | Profile-derived answers override stored ones and unstated owned questions are withheld. Sound |
| Packet | `applicationPacket.js` | Ownership check (name, email, profile, master resume fingerprint) runs in the runner itself. Content is not hashed (AF-10) |
| Field mapping | `formPrefill.classifyLabel`, `screeningQA.answerScreeningQuestion` | The two weakest links (AF-02, AF-03) |
| Scan | `domScanner.collectFields` | Native `input, textarea, select` only. No shadow DOM, no button-based widgets, no group labels (AF-06), native `required` only (AF-07), controls without id or name dropped (AF-01) |
| Fill | `fieldFiller.applyValue` | Sponsorship read-back and refusal are careful and worked in every probe. The typeahead path is not (AF-05, AF-09) |
| Validation | `detectValidationErrors`, `verifyFilledFields` | Recognises a fixed set of error markup; anything else reads as success (AF-08) |
| Review | `ready_for_submit` | Declared without the live required check (AF-07) |
| Submit | `tryAutoSubmit` | Eleven gates in order; the last one, verification, is the weak one (AF-04) |
| Tracking | `recordAssistResult` | Bypasses the state machine (AF-12); retraction is incomplete (AF-11) |

What is good and should be kept: the default stops before submit; an inverted sponsorship question is never answered; the sponsorship option is read back and cleared on conflict; a qualified "years of experience with X" question is never answered from the career total; uploads are matched by label and verified against the file list; the submit click refuses name-based fallbacks; a write-ahead checkpoint prevents a second click.

### 4.2 Rule compliance

| Rule | Result |
|---|---|
| Answers come from the authoritative source | Fails for misclassified labels (AF-02) and over-matched patterns (AF-03) |
| Unknown answers stay blank | Fails where a pattern over-matches (AF-03) and where a typeahead commits another element (AF-05) |
| Sensitive and demographic questions are never inferred | EEO questions (gender, veteran, disability, age) returned `ask_user` in every probe. Fails for citizenship (AF-02) and accommodation (AF-03) |
| Existing submit gates preserved | Hold on the designed path. Bypassed by AF-01; weakened by AF-04 |

### 4.3 ATS readiness matrix

Built only from what the repository proves. "Fixture" always means a hand-written imitation; no captured real page exists in `tests/node/fixtures`. `config/verified_live_ats.json` is empty and `docs/SUPPORT_MATRIX.md:253-257` says no ATS has a recorded supervised run. Earlier supervised runs are mentioned in test comments and session notes, but none is recorded where the gate reads it, so none is counted here.

| ATS | Detected | ATS-specific fill logic that runs | Fixture | Multi-step fixture | Submit clicked on a real DOM in tests | Live run recorded | Auto-submit today | Readiness on this evidence |
|---|---|---|---|---|---|---|---|---|
| Greenhouse hosted | Yes | Upload by label; `.select__option` | Yes, single page | No | No | No | Refused | Fixture-level, text and upload only. Typeahead questions likely block (AF-09) |
| Greenhouse embedded | `gh_jid` or frame URL | Frame scope | Yes, matches by file name | No | No | No | Refused | Fixture-level |
| Lever | Yes | None | Yes, single page | No | No | No | Refused | Fixture-level |
| Ashby | Yes | None | Yes, single page, no button-style yes/no | No | No | No | Refused | Fixture-level; AF-05 pattern resembles its markup (Unverified live) |
| Workday | `myworkdayjobs.com` only | "Apply Manually", dialog handling, announcement filter | Yes | Dialog and sign-in wall only | No | No | Refused | Reaches the sign-in wall in fixtures; form steps unproven |
| SmartRecruiters | Yes | None | Yes, single page | No | No | No | Refused | Fixture-level; shadow DOM unsupported (Static) |
| Oracle Recruiting | No (generic) | Honeypot, styled checkbox, cookie dialog; post-submit URL for the watcher | Inline imitations in tests | Yes, 2 steps | No | No | Never | Generic path with the most realistic fixtures |
| iCIMS, Taleo, Jobvite | Yes | None that runs | No | No | No | No | Never | No evidence |
| Rippling, Workable, BambooHR, SuccessFactors | No | None | No | No | No | No | Never | No evidence |
| LinkedIn, Indeed, Dice | No | Guard text only | No | No | No | No | Never | Not supported by design |
| Generic | Yes | The whole scanner | Yes: select, checkbox, radio, combobox with listbox, conditional | Validation-error case | No | n/a | Never | Fixture-level, plus the defects in section 3.1 |

Two documentation mismatches follow from this: `docs/SUPPORT_MATRIX.md:243-247` marks "Multi-step nav" and "Verified submission" as supported for five ATSes whose fixtures are single-page and whose submit is exercised only against an in-memory fake page; `README.md:244` lists the fixture gate but not the live-verification gate that currently refuses everything.

---

## 5. Ranking audit

### 5.1 Method

- Three personas from `tests/node/fixtures/personas.js` (fictional): ICU registered nurse (6 years, RN, CCRN, BSN), senior accountant (7 years, CPA, BS Accounting), backend software engineer (5 years, no education listed). Each was saved and activated in a throwaway workspace, so scoring used the same derived configuration a real user gets.
- 42 synthetic postings, 14 per persona, written for this audit and not taken from the test suite. Each has a relevance label from 0 to 3 assigned by me before scoring: 3 strong fit, 2 reasonable, 1 marginal or stretch, 0 should not apply.
- Each posting went through the pipeline's own sequence: exclusions, sponsorship classification, `scoreJob`, title verdict, tier.
- Fixed clock (2026-11-11 18:00 UTC), postings 2 days old unless stated.
- Limits: one labeller, small n, synthetic text. These numbers show behaviour on designed cases; they are not an accuracy estimate for real postings.

### 5.2 Results per posting

Tiers: Priority 85+, Apply 75+, Queue 60+, Near miss 50+.

**Nurse**

| ID | Label | Title | Score | Tier | Note |
|---|---|---|---|---|---|
| n1 | 3 | Registered Nurse - ICU | 90 | Priority | |
| n2 | 3 | Critical Care Registered Nurse | 68 | Queue | Missed fit |
| n3 | 3 | Staff Nurse - Intensive Care Unit | 75 | Apply | Read as "seniority +1" |
| n4 | 2 | Charge Nurse - ICU | 88 | Priority | |
| n5 | 0 | Nurse Practitioner - Critical Care | 88 | Priority | MSN and NP licence required (RK-01) |
| n6 | 1 | Registered Nurse - ICU (Clinical Nurse IV), 10 years required | 82 | Apply | Ineligible, penalty -8 |
| n7 | 1 | New Graduate RN Residency - ICU | n/a | Excluded | role_mismatch |
| n8 | 0 | Nurse Recruiter | 56 | Near miss | Title rejected |
| n9 | 0 | Travel RN - ICU (13 weeks) | n/a | Excluded | employment_mismatch |
| n10 | 3 | Registered Nurse - ICU, no description | 50 | Near miss | RK-04 |
| n11 | 0 | Medical Assistant | 57 | Near miss | |
| n12 | 1 | Registered Nurse - NICU | 87 | Priority | Candidate skills only under Preferred (RK-02) |
| n13 | 1 | Director of Nursing - Critical Care | 28 | Discard | |
| n14 | 2 | Registered Nurse - Emergency Department | 88 | Priority | |

**Accountant**

| ID | Label | Title | Score | Tier | Note |
|---|---|---|---|---|---|
| a1 | 3 | Senior Accountant | 93 | Priority | |
| a2 | 3 | Accounting Manager | 92 | Priority | |
| a3 | 1 | Staff Accountant (0-2 years) | 90 | Priority | "seniority +1 (step up)" (RK-03) |
| a4 | 1 | Senior Accountant, 12 years required | 82 | Apply | Ineligible, penalty -8 |
| a5 | 0 | Corporate Controller, MBA required | 49 | Discard | |
| a6 | 1 | Senior Tax Accountant | 90 | Priority | Candidate skills only under Preferred (RK-02) |
| a7 | 0 | Accounts Payable Clerk | 44 | Discard | |
| a8 | 0 | Account Executive - Accounting Software | 70 | Queue | Sales role, keyword overlap |
| a9 | 3 | Senior Accountant, no description | 53 | Near miss | RK-04 |
| a10 | 0 | Senior Accountant (3-month contract) | n/a | Excluded | employment_mismatch |
| a11 | 2 | Senior Accountant, 40 days old | 89 | Priority | |
| a12 | 1 | Senior Financial Analyst - FP&A | 54 | Near miss | |
| a13 | 2 | Senior Auditor | 60 | Queue | |
| a14 | 1 | Bookkeeper | 44 | Discard | |

**Software engineer**

| ID | Label | Title | Score | Tier | Note |
|---|---|---|---|---|---|
| s1 | 3 | Senior Backend Engineer | 93 | Priority | |
| s2 | 3 | Software Engineer, Backend | 88 | Priority | |
| s3 | 1 | Staff Software Engineer, 10 years required | 82 | Apply | Ineligible, penalty -8 |
| s4 | 1 | Junior Software Engineer | n/a | Excluded | role_mismatch |
| s5 | 1 | Frontend Engineer | 72 | Queue | Stack only under Nice to have (RK-02) |
| s6 | 1 | Backend Engineer (Java, Oracle) | 50 | Near miss | |
| s7 | 0 | Software Engineer, TS/SCI required | n/a | Excluded | security_clearance |
| s8 | 0 | Research Software Engineer, PhD required | 90 | Priority | Eligibility unknown (RK-01) |
| s9 | 0 | Sales Engineer | 57 | Near miss | Title rejected |
| s10 | 3 | Backend Engineer, no description | 50 | Near miss | RK-04 |
| s11 | 0 | Data Entry Clerk | 28 | Discard | |
| s12 | 2 | Site Reliability Engineer | 69 | Queue | Title verdict reject |
| s13 | 1 | Engineering Manager, Backend | 66 | Queue | |
| s14 | 0 | Senior Backend Engineer (3-month contract) | n/a | Excluded | employment_mismatch |

### 5.3 Aggregate

| Measure | Nurse | Accountant | SWE | All |
|---|---|---|---|---|
| NDCG@5 by score | 0.565 | 0.778 | 0.682 | |
| NDCG@10 by score | 0.733 | 0.904 | 0.844 | |
| Label 3 placed in Apply or Priority | 2 of 4 | 2 of 3 | 2 of 3 | 6 of 10 |
| Postings in Apply or Priority | 7 | 6 | 4 | 17 |
| of which label 3 | 2 | 2 | 2 | 6 (35%) |
| of which label 2 or 3 | 4 | 3 | 2 | 9 (53%) |
| Label 0 at Queue or above | n5 | a8 | s8 | 3 of 13 |
| Label 0 removed by a hard filter | n9 | a10 | s7, s14 | 4 of 13 |

### 5.4 Hard filters versus soft penalties

| Rule | Type | Behaviour measured |
|---|---|---|
| Employment type not accepted | Hard | Contract postings excluded for all three personas |
| Security clearance | Hard | s7 excluded |
| Junior and new-graduate titles for a senior search | Hard | n7, s4 excluded |
| Years floor unmet by 3 or more | Soft, -8 raw | n6, a4, s3 all stayed in Apply at 82 |
| Degree unmet | Soft, -8 raw, only when the candidate's degree is known and the pattern matches | a5 fell to Discard mainly on other factors; s8 unpenalised |
| Licence or certification unmet | Not modelled | n5 at 88 |
| Preferred versus required skill | Not modelled | n12, a6, s5 |

An ineligible verdict costing 8 points cannot move a 90 below the Apply line at 75, so the penalty changes order inside a tier but rarely the tier.

### 5.5 Stability

| Probe on the ideal posting of each persona | Result (identical across personas unless noted) |
|---|---|
| Same input twice | Identical score and tier, 42 of 42 |
| Upper-case body, CRLF, bullets rewritten as sentences, company-history blurb, tuition-benefit blurb, sections reordered | 0 points change in all 18 runs |
| Title variants: "(Remote)", requisition suffix, "II", "Sr.", inverted word order | 0 points change in all 14 effective runs (the "Sr." variant does not apply to the nurse title) |
| Age 0, 1, 3, 6, 7 days | No change |
| Age 14 / 30 / 44 days | -2 / -4 / -4 |
| Unknown posted date | -2 |
| Aggregator source instead of direct ATS | -3 |
| Unknown location / remote | -2 / +1 |
| Description removed or replaced by "See our website" | -40, Priority to Near miss |

Scoring is stable to formatting and boilerplate, which is a real strength. The single large instability is missing text (RK-04).

### 5.6 Extraction and dedupe

| Check | Result | Misses |
|---|---|---|
| Years-of-experience floor, 28 labelled sentences | 26 correct | "5+ years of experience (8+ preferred)"; "Experience (years): 5" |
| Required degree, 14 labelled sentences | 10 correct | "BSN required"; "MSN required"; "JD or LLM required"; "Degree in Accounting, Finance or related field" |
| Seniority from title, 26 titles | 14 correct | See RK-03 |

| Dedupe case | Expected | Got |
|---|---|---|
| Same role, two cities, same text | merge | merge |
| Same title, different teams, long different descriptions, same source | keep both | keep both |
| Same title, different teams, short different descriptions | keep both | merge |
| Same title, different teams, long different descriptions, different sources | keep both | merge |
| Same posting through two sources, tracking parameters differ | merge | merge |
| "Sr. Accountant" vs "Senior Accountant", same text | merge | keep both |
| "Acme Corp" vs "Acme Corporation", same text | merge | keep both |
| Two employers, company missing, same title | keep both | keep both |
| RN day shift vs night shift requisitions, near-identical text | keep both | merge |
| "Accountant II" vs "Accountant III" | keep both | keep both |

---

## 6. UI/UX findings

Method: a throwaway workspace seeded with 42 fictional jobs (software engineer, registered nurse, accountant) covering every status, served on port 4747 with sweeps and enrichment off; Chromium only; every non-localhost request aborted; run, auto-fill, prepare and link-check requests aborted in the page so nothing left the machine. 24 screens (7 main views, 17 dialogs) at 1440x900 and 390x844, light theme, with dark theme on 7 of them: 66 scanned states. The browser work was done by a sub-agent; I re-read the code behind UX-01 to UX-04 and UX-12 and viewed a sample of its screenshots. Score explanations and requirement lists in this workspace were seed text, so wording of the "Why" card is not evidence about the scorer.

### 6.1 The four questions

| Question | Answer | Evidence |
|---|---|---|
| Is the next action obvious? | On Today, yes: one card, one sentence, one primary button. In the job detail, often no: two primary buttons on most scored jobs, the action block starts about y=485 at 1440x900 behind a global "Next" panel, and for 14 of 42 jobs no primary button of the open job was inside the viewport. Rows show a status word, not an action | UX-05, UX-06 |
| Is the ranking explained? | Partly. Grade, score, "4 of 6 skills", matched and unevidenced skill chips, and the hard-filter reason are shown. Required versus preferred is not shown, salary is never shown, and a hard-filtered job still carries a "poor fit 12/100" card beside its filter reason | UX-09, RK-02 |
| Is uncertainty shown honestly? | Mostly yes, and this is a strength: "Description not fetched ... this row has NOT been judged on fit", "Reported as submitted, but not confirmed yet ... NOT counted as submitted until you confirm", "closed by the employer". Two exceptions: a not-judged job is also labelled "weak fit ... Reject it" in the same panel, and a missing location is silently omitted | UX-08 |
| Can mistakes be recovered from? | Reject, archive and outcomes: yes (undo toast for 12 s, plus "Clear"). Unverified submission: yes in the interface, but see AF-11. Approve: no undo anywhere, single or bulk. A submission recorded with a confirmation number: no undo | UX-03, UX-04 |

Persistence, checked after each action against the page after reload, the API and the throwaway database: Approve, Reject, Reject then Undo, Archive, "I submitted it myself" with and without a number, "Yes, it went through", "No, it didn't", outcome set and undone. No mismatch in 10 of 10.

Failure honesty, with injected HTTP 500 and aborted requests on approve, decision, mark-submitted, outcome, confirm and batch approve: an error toast and unchanged or rolled-back state in 7 of 7. No false success. One aborted request switches the whole app to "Disconnected" although the server is healthy.

### 6.2 Findings, screen by screen

| ID | Sev | Screen | Finding | Evidence | Concrete fix |
|---|---|---|---|---|---|
| UX-01 | Medium | Job detail | After "Approve only", focus moves to "Fill in browser". Enter pressed twice about 250 ms apart sent `POST /approve` then `POST /assist` (the auto-fill; aborted by the test guard) | Confirmed in browser. `detail.js` renders the approved plan with the fill button first | After approve, move focus to the next job in the list, or to a neutral element; never to an action that opens a browser |
| UX-02 | Medium | Job detail | Rejected, archived, employer-closed and hard-filtered jobs still show "Do this next" with primary "Auto-fill application" | Confirmed in browser; `actionPlan()` has no branch for these states (`detail.js:835-960`) | Add branches: rejected or archived shows "Restore" only; closed or filtered shows the reason and no apply action |
| UX-03 | Medium | Job detail | Reject is not click-guarded: 5 fast clicks sent 5 requests, wrote 5 events and stacked 5 undo toasts. After 2 clicks, Undo restores "rejected" because the second click recorded "rejected" as the previous state | Confirmed in browser. `decide()` is optimistic and not in `BUSY_ACTIONS`; the row is re-rendered so the busy marker is lost (`mutations.js:77-99`) | Ignore a decision equal to the current one; keep one undo per job holding the original state; use the server's `/undo-decision` (DI-04) |
| UX-04 | Medium | Detail, bulk bar | Approve has no undo and bulk Approve has no confirmation. "Approved 10" included a job whose panel says a screening question still needs the user, shown only at the bottom of the panel | Confirmed in browser; no un-approve route exists (section 3.2) | Add "Unapprove"; show "1 question left" beside Approve and in the bulk summary; undo toast for bulk approve |
| UX-05 | Medium | Applications, Today | Remaining-work counts go stale: the "Needs review 10" tab stayed at 10 after all 10 were approved until a reload. Today says "10 applications waiting on you" (hard cap `TODAY_LIMIT = 10`, `data.js:2517`) while the Next panel totals 28; the 10 packets awaiting approval were not on Today | Confirmed in browser | Refresh counts from the mutation response; say "10 of 28" and add "show more"; include approvals in Today |
| UX-06 | Medium | Job detail | No "next job" control. Reviewing and approving 10 jobs took 23 clicks one by one, 5 with bulk, and 522 key presses by keyboard because the detail panel follows the whole list in tab order (53 Tab presses from the first row to "Approve only") | Confirmed in browser | A review mode: one job, fixed keys for approve, skip, reject, undo, auto-advance, remaining count (feature 12). Until then, an "Approve and next" button and a shortcut to jump between list and detail |
| UX-07 | Medium | Job detail, 1440x900 | Two primary-styled buttons on most scored jobs ("Auto-fill application", "Prepare resume"); submitted jobs show "Prepare resume" as primary; Today shows 6 primary buttons at once | Confirmed in browser | One primary per panel; collapse the global Next panel when a job is open |
| UX-08 | Medium | Job detail | A job whose description was never read shows "weak fit" and "Low fit, probably not worth your time. Reject it" next to "Not judged yet" | Confirmed in browser; same cause as RK-04 | For not-judged jobs show only "Fetch description" and "Open posting" |
| UX-09 | Low | Rows, detail | Salary is never displayed and cannot be filtered; near-duplicate postings from one company appear as unrelated rows, both offering auto-fill | Confirmed in browser (seeded pay did not appear in any of 42 rows or panels) | Show posted pay when known and "pay not stated" otherwise; mark siblings "similar role at this company" |
| UX-10 | Low | Submitted job | Reject is accepted on a submitted, interviewing application; the row still says Submitted but the rejected total rises. "I submitted it myself" is still offered after a submission is recorded | Confirmed in browser | Hide decision actions once submitted |
| UX-11 | Low | Phone, 390 px | The status strip scrolls sideways inside its container, leaving the red "2 unverified submits" chip off-screen with no cue. Several targets are under 24 px tall (row title links 22 px, card titles 18 px, some checkboxes 20 px) | Confirmed in browser (emulated touch) | Wrap the strip; raise minimum target height to 24 px |
| UX-12 | Low | CLI | `node src/index.js dashboard --port 4747` ignored the port and tried 4646; only `--port=4747` or `DASHBOARD_PORT` is read (`index.js:372`) | Confirmed (sub-agent run), code re-read | Accept the space-separated form or reject it with a message |
| UX-13 | Low | Today | `/` does nothing on Today; with slow responses the list is blank with "Loading" only in the status bar | Confirmed in browser | Skeleton rows; make `/` switch to Jobs and focus search |

### 6.3 Accessibility and keyboard

| Check | Result |
|---|---|
| axe-core, WCAG 2.0 / 2.1 / 2.2 A and AA plus best practice, 66 states | 0 critical, 19 serious records (38 nodes), 4 moderate, 0 minor. All 17 dialogs clean in both themes |
| Rules hit | `color-contrast` (7 nodes: "Verify submission" chip 4.41:1, tailored-resume chip on a selected row 4.49:1, faint text on a selected row in dark theme 4.22:1); `label-content-name-mismatch` (truncated role link, Move menu buttons); `target-size` (one row checkbox under the sticky header); `landmark-unique` (two regions share one label) |
| Horizontal page overflow at 390 px | None on 24 screens |
| Skip link | Present, first in tab order, moves focus to main |
| Focus visibility, first 30 tab stops, both viewports | All visible |
| Dialogs (17) | Focus trapped, Escape closes, focus returns to the opener |
| Shortcuts | `/`, `?`, Esc, arrow keys and Enter on rows, `x` to tick a row. None for approve, reject or next job. Discoverable only through `?` or the profile menu |
| Console errors in normal use | 0 |

Not covered: WebKit and Firefox; a real screen reader; real touch devices; drag and drop on the applications board; master resume upload; profile switching; Runs and Inbox with data; dark theme beyond 7 screens.

---

## 7. Status of each prior-audit finding (2026-11-03, commit 1684886)

| Prior finding | Status | Evidence |
|---|---|---|
| Personal data in public git history | Still open | The archive repository is public and its history fails the personal-data check (SEC-01). The new public repository is a clean history. The friend's old fork returns 404; his current fork is of the clean repository |
| Windows CI fails on LF assumptions | Fixed | `.gitattributes` normalises line endings; public CI at `62b98b7` is green on all four Windows cells. The run before it failed on one Windows cell; the fixing commit is a browser-timing change, not line endings, so some Windows flakiness remains |
| Dedupe key is company + title | Partly fixed | Requisition-aware split exists in the working tree; 5 of 10 constructed cases still differ from the expected decision (RK-06) |
| Auto-submit verified on fixtures only; Oracle unproven | Still open, contained | No test clicks a real submit button; the live-verified list is empty, so auto-submit is refused everywhere. Oracle discovery is enabled and documented as producing rows; Oracle applications use the generic path with inline imitations only. AF-04 shows the verification step itself is unsound |
| SSRF guard checks address shape only | Fixed in the working tree, not in this checkout's HEAD | `safeFetch.js` pins the resolved address per hop. Vendor adapters still use plain `fetch` with configured hosts (low risk). The file is untracked here (SEC-02) |
| Oversized modules | Worse for three of four | `server.js` 3356 to 2284 lines (routes split out). `data.js` 3070 to 3164, `tailor.py` 2262 to 2424, `db.js` 1632 to 1661 |
| LLM layer does one optional task behind one provider | Fixed | Six tasks and four providers, all off by default. The privacy statement no longer matches what the new tasks send (SEC-07) |
| PR #3 unmerged | Fixed | Merged 2026-11-11 in both repositories |

---

## 8. Feature proposals, ranked

Each must respect the standing constraints: deterministic scoring by default, no fabricated qualifications, the existing submit gates, local-first, native Windows, every LLM feature opt-in. Dossier has no form filling, so none of the autofill ideas exists there.

| # | Idea | Problem | Design | Complexity | Privacy | Dossier / JobAgent today | Experiment that would justify it |
|---|---|---|---|---|---|---|---|
| 1 | Field-level provenance and confidence view | The user cannot see why a value was typed, which is how AF-02 and AF-03 stay invisible | For every fill, record source (profile field, answer id, pattern matched), match kind (exact, phrase, scattered), and the read-back. Show a table before hand-off: field, value, why, confidence; low-confidence rows start unchecked | M | Local; values already on disk | Neither. JobAgent records source and action per field but does not show it | Replay the 37-label and 22-question probes: share of wrong fills a reviewer catches with and without the view |
| 2 | Preflight simulator with a readiness report | Problems surface only after a browser has typed into a live form | Use the existing `dryRun` path: open, scan, plan, type nothing. Report planned values, blanks, required blanks, unknown widgets, upload slots, the submit control found. Result is ready, needs answers, or not automatable | M | Local; one page load per job, same as today | Neither | On a fixture corpus: proportion of later `assist_blocked` outcomes the preflight predicts |
| 3 | Approval bound to content | AF-10 | Digest at approval; "what you approved vs what will be sent" diff; block on difference | S to M | Hashes only | Dossier freezes a copy at send; JobAgent freezes versions but does not compare | Count approvals invalidated by later edits over two weeks of real use |
| 4 | Question bank keyed by intent | AF-03; every new phrasing is a new pattern | Normalise each question to an intent with attributes (topic, polarity, country, time frame). Answers attach to intents. Unknown intent means blank. Ship a regression corpus of question texts collected locally | M | Local; corpus holds questions, not answers | Neither | Precision and recall of intent matching on 200 collected questions against today's matcher |
| 5 | Safe resume of interrupted applications | AF-08, AF-11, AF-12 | Step fingerprint in the checkpoint; resume only if the live page matches; human-only retraction; `assist_started` written at start | M | Local | JobAgent has checkpoints; the gaps are listed above | Kill the process at each phase on fixtures; measure correct resume or correct refusal |
| 6 | Per-ATS reliability ledger | Readiness claims rest on documents | Local counters per ATS and host: runs, blockers by kind, fields accepted and rejected, hand-offs confirmed. Feeds the support matrix and warns before starting on a host with poor history | S | Local counts, no content | Neither | After 100 assisted runs: does the ledger's ranking of hosts predict blocked runs |
| 7 | Requirement table in the ranking explanation | RK-01, RK-02 | Extract required and preferred items; show met, not met, unknown with the quoted evidence; cap the tier when a required credential is unmet or unknown | M | Local | Dossier scores with a model; JobAgent has reasons for years and degree only | Re-run the benchmark: label-0 at Queue or above should fall from 3 to 0 without lowering NDCG |
| 8 | Fixture recorder | The readiness matrix has no real pages | One click in the assist window saves a sanitised DOM snapshot (values stripped, scripts removed) to a local fixtures folder; a script replays the scanner against it | M | Snapshot may contain employer text; values are removed; stays local | Neither | Number of new scanner failures found per 10 recorded forms |
| 9 | Stale and suspicious listing signals | Time spent on dead or bait postings | Add repost count for the same requisition, "open for more than N days", evergreen wording, and scam markers (personal email domain, payment requests, chat-app interviews). Show as badges, never auto-discard | S to M | Local | JobAgent has liveness checks and freshness; no repost or scam signals | Label 200 stored postings by hand; measure badge precision |
| 10 | Company-level duplicate guard | Applying twice to one employer for near-identical roles | Before prepare, warn when the same company was applied to within N days for a similar title; link to the earlier application | S | Local | JobAgent guards by URL and job id only | Count warnings that the user accepts as correct over a month |
| 11 | Outcome calibration report | Scores are not checked against results | Keep `scoring:calibrate` as a report: outcome rate by score band with confidence intervals; never change weights automatically; require a minimum sample per band | S | Local | JobAgent already has `calibration.js` (30 outcomes minimum) | Only worth extending once 100+ outcomes exist; until then show the sample size and nothing else |
| 12 | Keyboard review queue | Section 6 | One screen, one job, fixed keys for approve, skip, archive, undo; a visible count of what remains | M | None | Neither | Time to review 20 jobs before and after, measured on the seeded workspace |

**Rejected**: model-generated answers to unknown screening questions (breaks the blank-when-unknown rule); automatic handling of CAPTCHA or sign-in (forbidden); auto-submit on generic sites (no evidence base); learned ranking as the default (breaks determinism and has no data yet); cloud sync (breaks local-first).

---

## 9. Roadmap

Complexity: S under a day, M one to three days, L about a week.

### P0: privacy, security, data loss, broken core flows

| Item | Complexity | Depends on | Acceptance criteria | Tests |
|---|---|---|---|---|
| Make the archive repository private and request a purge (SEC-01) | S, owner action | None | Anonymous request to the repository returns 404; personal-data check passes on every public ref | Existing CI check |
| Commit or retire this checkout's working tree (SEC-02) | S | Owner decision | `git status` clean; HEAD contains the token and SSRF guard | Baseline suites on the committed tree |
| Stop clicking "Apply" on unscannable forms (AF-01) | M | None | R7 ends blocked with zero clicks | New fixture test |
| Label classifier precision (AF-02) | S | None | All 37 probe labels classify as listed in the fix; citizenship never filled from location | Table test, R4 |
| Screening matcher scope and polarity (AF-03) | M | None | All 22 probe questions give the expected result; shipped patterns pruned | Corpus test |
| Typeahead option scope (AF-05) | M | None | R5 records no click outside the field | Fixture test |
| Submit verification baseline (AF-04) | S | None | R1 ends `submit_unverified` | Fixture test with a real button |

### P1: correctness of "ready" and of state

| Item | Complexity | Depends on | Acceptance criteria | Tests |
|---|---|---|---|---|
| One required-field definition; live check before `ready_for_submit` (AF-07) | S | None | R3 blocked, both questions named | Fixture |
| Group labels (AF-06) | S | None | R2 scan returns the legends | Scan test |
| Progress check after advance (AF-08) | S | None | R6 blocked after one click | Fixture |
| Typeahead verification and option mapping (AF-09) | M | AF-05 | R9 and R9b end ready with the choice recorded | Fixture |
| Approval digest (AF-10) | M | None | R10 blocked with the changed answer named; `restore-version` resets approval | Unit and route |
| Retraction resets the checkpoint (AF-11) | S | None | R11 opens the form | Route |
| Assist results through `setJobStatus`; start marker (AF-12) | M | Transition table update | Illegal transitions rejected; event per change | Unit |
| Rejected packets and archived jobs are not filled (AF-13) | S | None | Batch skips with a reason | Route |
| Approval audit events and state check (DI-01), re-prepare guard (DI-02) | S | None | Events present; 409 on submitted | Route |
| Schema downgrade guard (SEC-05), launcher token probe (SEC-03) | S | None | Newer schema refused; token not sent to an unidentified listener | Unit |
| Detail panel next-action fixes: focus after approve, no apply action on rejected, closed or filtered jobs, guarded Reject, Unapprove (UX-01 to UX-04) | M | DI-01 | Double Enter after approve sends one request; rejected job offers only Restore; 5 fast Reject clicks send 1 request | Browser tests |

### P2: ranking and evidence

| Item | Complexity | Depends on | Acceptance criteria | Tests |
|---|---|---|---|---|
| Required credentials and tier cap (RK-01) | M | None | n5 and s8 below Apply; NDCG@5 not lower on any persona | Benchmark as a test |
| Required versus preferred weighting (RK-02) | M | None | n12, a6, s5 each at least one tier lower | Paired postings |
| Seniority per role family (RK-03) | S | None | At least 22 of 26 titles | Table |
| Needs-description state (RK-04) | S | None | n10, a9, s10 shown as not judged | Benchmark |
| Queue order (RK-05), dedupe normalisation (RK-06) | S each | None | Scenario NDCG equals score order; 8 of 10 dedupe cases | Unit |
| Provenance view, preflight (features 1, 2) | M each | P0 matcher fixes | See experiments | Browser tests |
| Live counts, one primary action, review mode (UX-05 to UX-07, feature 12) | M | P1 detail fixes | Counts correct without reload; 10 approvals in 20 key presses or fewer | Browser tests |
| Contrast, target size and label fixes (section 6.3) | S | None | axe reports 0 serious on the 66 states | Existing `a11y.test.js` extended |
| Documentation corrections (SEC-07, support matrix, README gate list) | S | None | Documents match section 4.3 | Doc test |

### P3: reach and upkeep

| Item | Complexity | Depends on | Acceptance criteria | Tests |
|---|---|---|---|---|
| Fixture recorder and replay (feature 8) | M | None | Ten recorded forms replay in CI | Replay test |
| Supervised live verification of one ATS, then record it | M, supervised | P0 and P1 complete | One entry in `verified_live_ats.json` with date and outcome | Manual protocol |
| Shadow DOM and button-based widgets in the scanner | L | Recorder | Recorded SmartRecruiters and Workday forms scan fully | Replay |
| Split `data.js`, `db.js`, `tailor.py` | L | None | No file over 1500 lines; suites green | Existing suites |
| Reliability ledger, question bank, duplicate guard, listing signals | S to M each | P1 | See experiments | Unit |

---

## 10. Quick wins

1. Remove `will you require`, `per month` and `how many years` from the shipped answer patterns (AF-03).
2. Add the disqualifier nouns to `classifyNameField` and route "citizenship" to the sensitive path (AF-02).
3. Delete `[class*="option"]` from `OPTION_SELECTORS` when no listbox is known (AF-05).
4. Call `unfilledRequiredControls` before returning `ready_for_submit` (AF-07).
5. Read `legend` in `labelFor` for radio and checkbox inputs (AF-06).
6. In `retractSubmissionReport`, reset the checkpoint (AF-11).
7. Refuse re-prepare when status is `submit_unverified` (DI-02).
8. Refuse to open a database whose `user_version` is above the code's (SEC-05).
9. Add `JobAgent/` to the eslint ignore list or delete the nested clone (SEC-11).
10. Correct `SECURITY.md:34` and the README gate list (SEC-07, section 4.3).

---

## 11. Verification summary

| Area | Tested how | Passed | Failed | Unverified |
|---|---|---|---|---|
| Existing suites | verify, lint, typecheck, vitest, pytest, once each | All | None | Coverage thresholds, ruff, Windows and Linux locally |
| Autofill | 11 local fixtures through `assistApply` with real Chromium; 37-label and 22-question probes | Sponsorship truthfulness, inverted questions, EEO left blank, upload handling (by reading and existing tests) | R1, R2, R3, R4, R5, R6, R7, R9, R9b, R10, R11 | Behaviour on any real ATS; AF-16 |
| State transitions | Code map by a sweep agent; I re-read the paths behind AF-11, AF-12, AF-13, DI-01, DI-02 and reproduced AF-11 | n/a | As listed | DI-03 run, DI-06, concurrency across processes |
| Ranking | 42-posting benchmark, 3 personas; 68 labelled extraction items; 10 dedupe cases; 75 stability probes | Determinism, formatting stability, hard filters for employment type and clearance | RK-01 to RK-07 | Real-posting accuracy |
| UI | 66 rendered states in Chromium, axe-core, keyboard walks, 10 persistence checks, 7 injected failures, 5 double-click checks (sub-agent; code behind UX-01 to UX-04 and UX-12 re-read) | Persistence 10 of 10, failure honesty 7 of 7, dialogs, skip link, focus visibility, no page overflow at 390 px | UX-01 to UX-13; 19 serious and 4 moderate axe records | WebKit, Firefox, screen reader, real touch, drag and drop |
| Security | Sweep agent; I re-read the code behind SEC-03, 04, 05, 07, 08, 09 and ran the repository visibility and history checks | Bind address, token compare, origin checks, CSP, SSRF guard for user URLs (sweep) | SEC-01 to SEC-11 | Export staging permissions, Windows ACLs, extension frame trust |

**Needs supervised live validation** (a person watching, test accounts, no real submission unless the owner chooses): one end-to-end assisted fill per ATS in the allow list, to replace imitation fixtures with recorded ones; whether AF-05 occurs on Ashby and AF-09 on Greenhouse; whether AF-01 occurs on any signed-in quick-apply page; Windows start-up from a path with spaces and without Python on PATH.

---

## Appendix A. Reproductions

All runs: `assistApply` from `src/automation/applyRunner.js`, real headless Chromium, `JOB_AGENT_ROOT` set to a throwaway root, candidate and answers from `tests/node/fixtures/config/` (fictional), an in-memory state store, `ownershipCheck` stubbed to pass, packet `{ job: { id, url: file://... }, fillPlan: { steps: [], uploads: [] }, screeningPlan: { answered: [], needsUser: [] }, blockers: [] }`.

| R | Page content (essentials) | Options | Observed |
|---|---|---|---|
| R1 | Required email input; paragraph "Once your application is submitted, you will not be able to edit it."; `<button id="submit_app" type="button">` whose handler only writes an error message | `autoSubmit: true` | `status: submitted`, `submitted: true`, signals `["confirmation message"]`, phase `verified`; URL unchanged, form still present, page error visible |
| R2 | Email; two `fieldset > legend` radio groups (relocate, background check) with wrapped labels Yes / No; a checkbox fieldset "Which shifts can you work?" | default | Scan labels: `radio:Yes`, `radio:Yes`, `checkbox:Day`, `checkbox:Night`; nothing checked; `ready_for_submit` |
| R3 | Email, textarea "Why do you want to work at Acme Hospital? *", text "Nursing license number *", all `aria-required="true"` | default | Both questions listed as optional and left blank; `ready_for_submit` |
| R4 | Labelled inputs: First, Middle, Last name, Email, Current company name, Reference name, Reference email, Emergency contact phone; selects: Country of citizenship, "Have you ever failed a background check or drug screen?", "Will you require any accommodation..."; text "Will you require relocation assistance?" | default | Middle, company and reference name received the full name; reference email the candidate's email; emergency phone the candidate's phone; citizenship "United States"; background "Yes"; accommodation "Yes"; relocation the sponsorship sentence; `ready_for_submit`, no questions raised |
| R5 | Email; a question "Do you now or in the future require visa sponsorship?" with two `<button class="_option_...">` Yes / No that log clicks; `<input role="combobox" aria-autocomplete="list">` labelled Location with no listbox | default | Click log: `sponsorship:Yes`; Location reported filled; `ready_for_submit` |
| R6 | Email; `<button id="next">Next</button>` whose handler writes a message into a plain `div` and does not navigate; heading "step 1 of 4" | default | 7 clicks, 8 fills of Email, `advancedSteps: 7`, `ready_for_submit` |
| R7 | Two inputs wrapped in labels, no `id` or `name`; `<button id="apply">Apply now</button>` that replaces the form with "Thank you for applying" | default | Button clicked by the run; page shows the thank-you text; `assist_blocked: no application form found` |
| R9 | react-select imitation (input inside `.select__control`, options rendered as `.select__option[role=option]`, filtered by typed text, input cleared on commit) labelled "Are you willing to relocate?" | default | Sentence answer typed, no option matched, `assist_blocked` |
| R9b | Same widget labelled "Do you consent to a background check?" | default | "Yes" committed correctly, then `assist_blocked: was not accepted: value did not remain set` |
| R10 | Required Email and "Desired salary"; a submit button that navigates to a page reading "Thank you for applying" and echoing the salary | `autoSubmit: true`; packet plan holds the approved salary answer; live config holds an edited one | `submitted`; the edited salary was sent; nothing withheld |
| R11 | Any form | Checkpoint first marked with `markApplicationUserSubmitted` | `submit_unverified`, `duplicatePrevented: true`, nothing filled |

Probes (pure functions, no browser): `classifyLabel` on 37 labels; `answerScreeningQuestion` on 22 questions with the fixture answers and a candidate who requires sponsorship.
