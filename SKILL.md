---
name: j-QA
description: >-
  Manual, human-perspective QA of a change through the REAL running UI — not a
  code review. Scopes itself to the current git changes, opens the app in a
  browser, and walks each changed feature the way a domain expert / business
  analyst / everyday admin actually would: is the control where I'd look, is the
  panel easy to fill, does it work, is the calculation right, after I submit do I
  clearly know something important happened, does it persist when I come back and
  into the next cycle, and did it land correctly in the backend + audit trail.
  Derives the EXPECTED behaviour from the changed code itself (the oracle),
  designs scenarios from real personas, executes them in the browser with
  screenshot evidence, and ends in a SHIP / DON'T-SHIP verdict with a defect log.
  Invoke when the user says "j-QA", "QA this", "manually test the UI", "test it
  like a real user / domain expert", "does the UI actually work", or after
  building/changing any user-facing feature. Sub-agents run on the Sonnet model.
---

# j-QA — Manual UI QA, from a real tester's chair

A feature is not "done" because the code looks right and the tests are green. It
is done when a real person can **find it, use it, understand it, trust the number
it produces, come back later and still see that number, and prove it in the
backend.** This skill is that person — a domain-aware manual tester — walking
the actual product.

It is the sibling, not the twin, of `feature-impact-sweep` and
`adversarial-review`: those **read the code** to hunt bugs; j-QA **uses the
running app**. When regression across neighbouring code is needed, j-QA *calls*
the impact sweep rather than repeating it.

Grounded in recognised practice so the output is credible to any reviewer or
buyer: ISTQB test-design techniques (equivalence / boundary / state-transition /
decision-table), Nielsen's 10 usability heuristics (two-pass method, 0–4 severity
scale), and standard UAT discipline (entry/exit criteria, persona-based
scenarios, a prioritised defect log).

---

## The one rule

**Judge the product, not the report.** Never pass a feature because the diff
"looks correct" or a toast appeared. A finding is real only when you *saw it in
the UI* (or in the DB you queried). "It renders" is not a pass; "I entered X, the
screen showed Y, the DB row held Z, and the next cycle still used it" is.

## Adaptive by design (this is a hard requirement, not a nicety)

Run only the phases the change actually needs, and **write one line saying why**
for every phase you include or skip. A pure display tweak does not need the
cross-cycle persistence phase; a backend-only change may skip the browser
walkthrough. Never run a phase for ceremony, and never *silently* drop one —
a skipped phase with no stated reason is a bug in the QA pass itself.

## Execution model (why it's shaped this way)

- **The browser is a single shared pane.** Two agents cannot click it at once.
  So the *thinking* work fans out in parallel; the *clicking* work is serial.
  - **Parallel (Sonnet sub-agents):** reading the changed code to build the
    oracle, designing scenarios per persona, and verifying the database.
  - **Serial (one driver — the orchestrator or a single dedicated browser
    agent):** the actual UI walkthrough.
- **All sub-agents run on the Sonnet model** (`model: sonnet`). The orchestrator
  reconciles their output.
- **Evidence or it didn't happen.** Every executed scenario carries evidence and,
  where money is involved, the DB/ledger row it was checked against.
- **The verification ladder — and name the rung you reached.** Not every scenario
  can be clicked live (modals crash, data isn't set up). Prove each at the highest
  rung you can reach and *label it*: **(1) UI-live** (clicked it) > **(2) DOM**
  (`read_page`/`get_page_text` asserts the element/text exists) > **(3) API**
  (the endpoint returns the right code/body) > **(4) DB** (the row is correct) >
  **(5) code-oracle** (the logic provably does it). A gate proven at API+DB with
  the dialog un-clicked is a real pass — but it is a rung-3/4 pass, not rung-1;
  say so. Never dress a lower rung up as a live UI pass.
- **Driving the UI reliably beats driving it fully.** `screenshot` can hang while
  `read_page`/`get_page_text` still work — prefer the text/DOM tree for assertions
  and use screenshots only as spot evidence. If the in-app browser crashes on an
  interactive control (dropdown/modal), don't fight it: verify that flow at
  rung 3–4 (API/DB), record the UI-click as a gap with the reason, and recommend a
  Playwright script for the repeatable interactive suite. A crashed renderer is a
  tooling finding to report, not a feature defect.

---

## Phase 0 — Scope & test basis  *(always)*

1. Get the change set. Uncommitted work: working-tree diff. A branch:
   `git diff <base>...<head>` (base is usually `test` or `main`). List every
   changed and new file.
2. Build the **change-to-surface map** — the spine of the whole pass. For each
   change, fill: *what changed → which screen/route shows it → which API endpoint
   it calls → which table / ledger row it writes.* Group changes by feature area.
3. Derive the **oracle** (the expected behaviour) **from the code**, because there
   is usually no written spec. What is each field's rule, each validation, each
   calculation, each success/failure message the code will emit? This is what the
   UI will be checked against.
4. Decide which later phases apply; record the reason for each include/skip.
5. State **entry criteria** (build is up, data seeded, a login works) and **exit
   criteria** (every Blocker and Major defect closed or explicitly waived).

## Phase 1 — Environment & smoke gate  *(always)*

The entry gate. Do not design 40 scenarios for a screen that 500s.
- Confirm the stack is up and reachable, and you can log in as the needed role.
- Open each changed screen. Does it **render** with no red console errors and no
  failed network calls? Are the **new/changed controls actually present** and
  enabled where they should be?
- For any NEW page: confirm a sidebar/menu item exists and navigates to it — a
  page reachable only by URL fails the gate as "undiscoverable".
- If the gate fails: STOP, report exactly what broke, and go no further. A broken
  gate is itself the headline defect.

## Phase 2 — Scenario design  *(always)*

From the oracle + personas, derive the test scenarios. Fan out to parallel
**Sonnet** agents (one per persona, or per feature area) — **design only, no
browser yet**. Reconcile into one scenario matrix.

Personas (the three mindsets to test from — rename to match your domain):
- **Business Analyst** — "does this satisfy the business rule / acceptance
  criteria?" Owns the correctness-of-logic scenarios.
- **Power operator / admin** — "can I actually get my real, time-pressured job
  done with this?" Owns the real end-to-end flow, discoverability, and
  recovery-from-mistake scenarios.
- **End user** *(only if the change is user-facing)* — "is what I see about my
  own data clear and correct?"

For each feature area cover, using ISTQB techniques: the **happy path**;
**negative / invalid input** (blank, wrong type, over/under limit, forbidden
combination); **boundaries** (zero, first/last day, min/max, rounding edges);
**state transitions** (draft → submitted → reopened → deleted, etc.); and any
**cross-cycle** case (this period vs next period). Each scenario states its inputs
and its expected result taken from the oracle.

## Phase 3 — Real-flow UI walkthrough  *(always, for any UI change)*

Drive the **actual browser**, one scenario at a time. Use a two-pass method
(Nielsen): first complete the task as the persona would; then sweep the screen
against the lens below. For **every** scenario, answer these rungs — this is the
"how a real person tests" core. **ADHERENCE RULE: every rung gets an explicit
answer per scenario (PASS / issue / "not applicable because…"). A requester's
brief may deepen specific rungs; it never removes one. If time forces skipping a
rung, the report must say "rung N not checked" — silence counts as a false PASS.**
*(Canonical adherence miss: a reports-viewer gate emphasized downloads+numbers;
the pass nailed those but never ran rung 8, shipping tables full of raw UUIDs the
skill was explicitly designed to catch.)*

1. **Discoverability** — is the trigger control where a user would look, and easy
   to reach? Or is it buried / mislabelled / needing tribal knowledge?
2. **The panel / form** — when it opens, is it clear what to enter? Are labels,
   help text, defaults, and required-field markers understandable without asking?
3. **It works** — do the controls function; does validation fire on the *right*
   inputs (and *not* on valid ones); are there dead ends or stranded states?
4. **The calculation is right** — do the numbers on screen match the oracle to
   the rupee? (Cross-check against Phase 4's DB truth for money-bearing cases.)
5. **Feedback** — after submit, does the user *clearly* know something happened —
   and, when it affects something consequential, is that consequence spelled out (not a silent
   success, not a bare toast that vanishes)? The classic failure is "I clicked
   and nothing seemed to happen."
6. **Persistence on re-entry** — leave the screen and come back / refresh: is the
   change still there and still correct, or did it silently revert?
7. **Backend + ledger** — did it land correctly in the database, and is it
   visible and correct in the audit/ledger surface a user would check?
8. **Show names, not IDs** — every value shown to a user must be something a human
   recognises (a person's name, a record code, a cycle label), never a raw
   UUID or internal key. Flag any column, field, toast, or detail row that leaks an
   internal ID. (Nielsen: match the real world; recognition over recall.)
   *Canonical miss:* an audit ledger's "Who" column printed the actor's UUID
   while a neighboring column on the same row showed a proper name — the data
   had the name; the wrong field was bound.
   *Second canonical miss (generic viewers):* any table that derives its columns
   from the DATA's keys (union-of-row-keys report viewers, JSON-driven grids)
   will leak `id` / `*_id` columns by construction — enumerate EVERY rendered
   column and flag raw-key leaks. Acceptable treatments: human columns first;
   ids truncated + copy-icon (full value on copy/tooltip/export) or behind a
   "technical columns" toggle — never full-width raw UUIDs as reading matter.
9. **Can I actually get there? (menu reachability)** — every new page must be
   reachable by *clicking through the live menu from a fresh login*, not only by
   typing its URL. If URL-only works but no sidebar item appears, the route was
   registered without a visible navigation entry — a delivery gap, not a nicety.
   *Example gotcha:* a route registered in a route table makes the URL resolve;
   a user only finds it if it's ALSO registered in the navigation/menu catalog
   AND has a matching menu-entry seed/migration (or the nav click errors).
   Check every registration point a new page needs, not just the route.

10. **Friction sweep (the "would this annoy a professional daily-user" pass)** —
   after function is proven, re-read the screen as material: column ORDER (does
   the eye land on what matters first?), number formats (₹ separators, 2dp
   consistency, negative rendering), date formats (human, consistent), header
   labels (Title Case, no snake_case/jargon leaking from the API), empty-state
   wording (says what to do next, not just "no data"), truncation/wrapping of
   long values, units stated, totals visually distinct from data rows. List each
   friction item with its 0–4 rating — "worked but reads like a database dump"
   is a finding, not a pass.
   *"Reference-before-work" discoverability gap:* when a screen pairs read-only
   reference/context material (a legend, a lookup table, definitions) with the
   actual actionable content (a grid or form), and the reference material is
   unconditional — always full length, always rendered first — the actionable
   content's on-screen position degrades linearly with how much reference
   material exists. Invisible in a demo with 2–3 items, a real scroll-hunt at
   realistic scale. Don't just eyeball it on the QA fixture: measure (or
   estimate from row height × count) how far down the actionable content sits
   using the LARGEST realistic real-world row count available, and report the
   growth math (e.g. "≈45px/row × 20 rows ⇒ grid starts ~900px down"), not just
   a qualitative "feels buried." The same shape shows up sharper on an
   empty-state: if a screen can have nothing to act on, confirm that message is
   reachable WITHOUT first rendering unrelated live controls (date pickers,
   upload/save buttons) that have nothing to do — the empty-state message
   should be the first thing on the page, not the last.
   *(Canonical: a per-record override editor rendered a full "Field
   Reference" table of every available field above the actual editable grid,
   unconditionally — on a real record this pushed the grid to the bottom of a
   standard viewport at just 5 fields, and on a record with zero
   overridable fields, the "nothing to override here" message was the very
   last thing on the page, after a full screen of inert controls.)*

Rate every usability issue 0–4 (0 none, 1 cosmetic, 2 minor, 3 major/blocks the
task, 4 catastrophe). Capture a **screenshot** at each key step as evidence.

### Before you file a failure, rule out your own probe

A surprising share of "bugs" are the harness lying. Every automated *negative*
result — element missing, text absent, panel never appeared — must be confirmed
a second way before it reaches the report. The failures that keep recurring:

- **CSS `text-transform` breaks text matching.** `innerText` returns text as
  *rendered*, so a heading with an `uppercase` class comes back
  `GAINING THIS BY 2026-10-02`. A case-sensitive `/Gaining this by/` says the
  panel is missing when it is right there. **Always match page text
  case-insensitively.**
- **A loose selector regex grabs the wrong control.** `/Search/i` matched the
  global Ctrl-K command bar instead of the lens's own `Search capabilities…`
  picker; every later step then acted on nothing. Enumerate the real controls
  (`[...document.querySelectorAll('button,[role=combobox]')]` → text) and target
  the one you actually mean. Check the ARIA **role** too — a shadcn picker is
  usually `combobox`, not `button`, so `getByRole('button', …)` times out.
- **"Not rendered" is often "not reached".** A panel gated behind selecting a
  record cannot appear until you select one. Before filing, grep the source for
  the gating condition and confirm you satisfied it.
- **Empty-because-correct looks identical to empty-because-broken.** A
  projection panel with no rows may simply have no qualifying data. Seed a row
  that *must* appear, then assert it appears — that is the only version of this
  check worth anything.
- **A timeout is not a verdict.** One `page.goto` timeout on a server that
  answers `curl` in 0.25s was a network blip. Re-check liveness out-of-band
  before blaming the app.

Corollary: when a live result contradicts a source read, suspect the probe
first — but never *assume* it; prove which one is wrong.

## Phase 3b — Stateful & judgment checks  *(always, for any editor / approval screen)*

Phase 3 proves the app **reacts**. This phase proves it is **right after a
sequence**, and that the screen **makes sense to the person using it**. These are
the findings automation cannot see: a script detects a blank page, a 403, a
missing label — it does not notice that a counter is wrong or that an approver
has nothing to approve on.

**Hard rule: a screen is not passed until you have reversed every change you made
and asserted the baseline returned.** Testing only the forward path is the single
biggest blind spot in a UI pass.

*(Canonical origin: an Access-Rights sweep produced 18 findings — every one of
them a crash, a 401, a duplicate key or a missing label. A human reviewer then
found 12 more in one sitting, all of them state-after-a-sequence or
does-this-make-sense. The automated pass had tested that things responded, never
that the result was correct.)*

**A · Round-trip / reversal.** Apply a change, reverse it, assert the screen
returns to the *exact* baseline — counter back to 0, staged diff empty, nothing
left selected. Then the compound path: bulk-apply ("Allow all") → remove one item
→ re-apply the bulk action → assert the expected end state.
*Canonical miss:* a role-editor change counter incremented on every toggle and
never decremented — turning a permission ON then OFF again left "Review 1 change"
with nothing actually changed. Related: removing permissions individually left the
parent group still ticked, and re-selecting the group did not restore them
(a deny recorded by the manual removal kept shadowing the group's grant).

**B · Create-vs-Edit parity.** When a screen exists in both modes, diff them.
Anything that summarises *existing* data (impact preview, holder counts, "who is
affected") is meaningless in create mode and should be hidden or scoped;
anything present in one mode and absent in the other needs a stated reason.
*Canonical miss:* a "live impact" panel on the create-role screen permanently read
"0 people affected", and the review dialog's added/removed drill-down existed when
editing a role but not when creating one.

**C · Post-mutation refresh & selection identity.** After every save, ask two
questions: does the **list** update without a manual reload, and is the item you
just created/edited the one now selected/highlighted — not index 0? Verifying the
DB row changed is *not* enough; the list view is the downstream surface the user
actually reads.
*Canonical miss:* marking an item reviewed wrote correctly to the database and the
worklist never refreshed; and after creating a role, the picker selected the first
role in the list instead of the newly created one.

**D · Can the decision-maker actually decide?** On any approval / review /
worklist screen, adopt the approver's chair: from what is on this screen **alone**,
could they responsibly say yes or no? They need *what the thing does* (not just its
code or name), *who asked and why* (the recorded reason travels with the request),
and a *visual difference between adding and removing* (green/red, not two identical
cards). Also check the list scales: filtering or grouping once the queue is long.
*Canonical miss:* a review queue listed bare permission names with no description,
no requester reason, and identical styling for grants and revocations.

**E · Superseding / concurrent requests.** For any request → review → apply flow,
submit a change, then submit a *contradictory* change to the same object before the
first is approved. Which wins? Is the precedence documented and does the UI show it?
An undefined answer here is a spec gap worth reporting, not a bug to guess at.
If the second write succeeds silently, also check **where the record actually
landed** — re-query the DB for the invisible context it carries, above all which
parent grouping (cycle / batch / run / period) it attached to. Do not trust the
screen's own summary: the UI's counters can round-trip perfectly while the record
quietly attaches to the wrong parent, and no error appears anywhere.
*Canonical miss:* a salary-revision proposal created straight after a brand-new
cycle silently filed itself into an unrelated two-month-old cycle, because the
resolver preferred the first `active` cycle and a fresh cycle starts as `draft` —
there was no cycle picker in the wizard and no cycle column in the ledger, so the
screen could not have revealed it. Found only because the round-trip cleanup step
(pattern A) forced a check of where the data had really gone.

**F · Does this feature earn its place?** For each feature on the screen, name the
simpler alternative and state what this adds. If you cannot, that is a finding.
*Canonical miss:* a "copy this person's access" flow that copies only their roles
sits next to "pick roles directly" — worth keeping only for copy-by-example, and
worth saying so on screen.

**G · Cross-screen affordance parity.** Inventory the bulk actions and helpers on
each related screen and flag asymmetries. If one screen has "Allow all / Block all"
or a search box and its sibling doesn't, that is a defect of consistency.
*Canonical miss:* the role editor had per-group Allow-all/Block-all and the People
tab had neither; separately, the editor offered **no search at all** across 232
capabilities spread over 10 accordions (one holding 64).

## Phase 3c — The silent-failure family  *(always, for any flagged / multi-service / notifying feature)*

Phase 3 asks "does it work?". Phase 3b asks "is it right after a sequence?". **This phase asks the one
question that catches more real damage than either: *what would this look like if nothing had happened?*
If the answer is "the same", the signal is worthless — go and count rows.**

Each item is a bug CLASS with a check you can run in minutes. Every one has shipped past a full green
test suite somewhere, because the tests asserted a response and not an effect.

**A · The silent downgrade.** Any feature behind a feature flag, an entitlement, a licence check, or a
"context"/"actor" object has an `else` branch. **Find it and read it.** The failure mode is not the flag
being off — it is the new path being *skipped* because one input was missing, while the UI reports
success. Ask: *if the flag is ON but the caller's context is incomplete, what runs?* If the answer is
"quietly, the old behaviour", that is a defect whether or not you can trigger it.
*Shape to look for:* a context object built as `a && b && c && d ? {...} : undefined`, then used as the
gate for the new path — any one missing value silently selects the legacy branch.
*Seen:* a request saved as "pending approval" with **no approval record behind it at all**, under a toast
reading "submitted — someone will review it shortly". Nobody would ever have been asked.

**B · The guard inside the branch it guards.** When you find a rollback, transaction, or validation whose
comment describes exactly the bad state you are hunting — check **which branch it lives in.** A guard
inside the happy path cannot fire on the path that skips the happy path.
*Seen:* a comment reading "never leave a pending record with nothing to approve it" sitting inside the
very branch that only runs when things went right. It protected against the downstream call *failing*; it
could not fire when the code never made the call. The author anticipated the exact bad state, guarded the
loud path, and the silent path walked past it.

**C · A state the backend has and the UI never learned.** Enumerate every value of every status enum the
feature can produce, then grep the frontend for each one. **Any value with zero references is invisible
in the product.** Pay special attention to views that render from a *different* field than the one that
changed — a per-step diagram will not notice a record-level state change.
*Seen:* a "sent back for correction" state written correctly, with an honest audit row, while the progress
diagram had zero references to that value and drew from step rows still marked pending. The screen was
pixel-identical before and after. **Sending something back and doing nothing looked the same.**

**D · Did the person who now owes an action get told?** For any handoff across a module, service, or
system boundary, name the human who must act next and open **their** screen — not the screen of whoever
just acted. Confirm the state actually crossed the boundary.
*Seen:* one service held "needs correction" together with the reviewer's written reason, while the owning
module's record still read "pending approval". The requester's screen was unchanged in every respect —
**the only person who had to do something was the only person not told** — which also left the elaborate
correction logic downstream unreachable through the UI.

**E · Templates that declare variables they never use.** For every notification, email, SMS, or document
template: list its declared variables, then confirm each appears in the rendered subject or body. **A
declared-but-unused variable is a bug in the template**, and the symptom is generic, untriageable
messages. Then read the result as a queue: *could someone triage ten of these without opening any?*
*Seen:* a template declaring two variables — one used only to build the link, the other used nowhere —
producing a fixed sentence with no type, no amount, no requester. Two notifications days apart rendered
character-for-character identical. The data was handed to the template and thrown away.

**F · Generated documents get the full rung-8 treatment, plus encoding.** Exports escape the product, so
they matter more than a screen — and they are almost never opened during QA. **Open the file.** Check:
1. **Encoding** — can it render your currency symbol and a real dash? Many PDF libraries default to a
   Latin-1 font, and code that "safely" strips non-ASCII will substitute `?` for every one. A money
   document that cannot print its own currency symbol is a serious finding.
2. **IDs where names belong** — rung 8, applied to the file.
3. **Internal leakage** — any localhost/internal URL, debug string, or key=value diagnostic on a document
   that carries the company's legal identifiers.
4. **Truncation** — long values cut off instead of wrapping.
5. **Formatting consistency within the one document** — not two number formats on one page.
6. **Instance vs definition** — does it describe *this* record, or dump the whole configuration? If most
   of the page is definition, the specific facts get buried in what didn't happen.
*Seen:* an audit certificate that could not print the currency symbol, showed its routing decision as two
raw UUIDs, carried a localhost URL on a page bearing the company's tax identifiers, and spent 80% of its
length listing configuration branches the record never touched.

**G · Is the primary action on screen when the page opens?** Land on each actionable screen as the person
who must act, and ask whether the thing they came to do is visible **without scrolling**. A page whose
action bar sits below the fold reads as read-only, and someone arriving from a notification will bounce.
*Seen:* an approver saw header, summary and a progress diagram; the approve/reject controls sat far enough
down that the page looked purely informational. Context above, task below.

**H · The same fact twice, in two vocabularies.** Inventory adjacent columns and badges for duplication.
Two fields describing one state make the reader work out whether they are one thing or two — and they
*diverge* on edge cases, at which point the product contradicts itself on a single row.
*Seen:* a status chip beside a separate approval column saying the same thing in different words. On a
broken record the pair actively misled — the chip claimed a state the blank column correctly denied, and
**the blank column was the truthful one.**

**I · Intermittent ⇒ do NOT close on a passing repro.** When a silent-failure mechanism is *readable in
the source*, three passing attempts do not unprove it. Record the passes as data, keep the finding open,
and **fix the silent fallback before hunting the trigger** — the fallback is what makes the trigger
invisible. Then sweep for existing damage with a query, and **filter by date** so rows created before the
feature existed don't drown the signal.
*Seen:* the same path failed once then succeeded three times with no code change; a parallel session's two
repro attempts both passed and it was ready to report "cannot reproduce". The damage sweep returned 70 rows
of which 68 were pre-feature noise — the date filter was the difference between a real number and a scary
one.

**J · Prove which build answered — front end AND back end — before trusting ANY defect report, including
your own.** Two halves, and getting only the first is the trap:

*Front end:* a dev-server tab that outlived a restart serves whatever it last had. Hard-reload, and before
believing a component "never shipped", fetch the module from the dev server and grep for a symbol you
expect.

*Back end — the half that is usually skipped:* **when several branches or worktrees run their own backend
against the SAME database, the front end looks right and the write lands somewhere else entirely.** A branch
without the feature will save a record in the pre-feature shape, which is **indistinguishable from a product
defect** at the database. So: confirm the API base URL points at the backend you mean, then **perform one
write and read the network panel to see which host:port actually answered.** Never conclude a root cause
from a source read plus a database row when more than one build can write that row.

Also check **committed ≠ deployed**: compare the fix commit's timestamp against the running build's artifact
(`dist`) and grep the artifact for a symbol the fix introduced. Testing a fix against a stale process
produces a confident false "not fixed".
*Canonical miss:* an orphaned record was diagnosed from source as a missing-context fallback, complete with
a code path and a mechanism. The real cause was that the write never reached that stack at all — a sibling
branch's backend, sharing the database, lacked the integration entirely. The served **front end** had been
verified against the branch; **which backend answered the POST had not.** The mechanism found in source was
real but was not what happened.
*Seen:* a reported "the new panel isn't there" was a tab predating the commit, on a front end that had been
moved onto another lane's port to dodge a CORS mismatch — so "which build am I looking at?" was no longer
answerable from the URL. Twenty minutes lost and one false defect filed. **Fix a CORS mismatch by
overriding the setting, never by moving the front end onto a port that belongs to someone else.**

**Record what works, to the same evidence standard.** A log that is 100% defects loses the reader's trust
and invites someone to rebuild a working feature. When you watch something work end to end, write it down
with the rows or audit trail that prove it.

## Phase 4 — Cross-cycle persistence & backend truth  *(conditional)*

Run only if the change **writes data that a later process reads** (scheduled
runs, compliance/regulatory records, ledgers, running totals, carry-forward
balances). Skip with a one-line reason for display-only or read-only changes.
- Query the DB directly for the rows the change should have written; confirm
  values, `who/when` metadata, and soft-delete flags — not just that a row exists.
- Advance to **the next cycle/period** and confirm every designed case still
  computes correctly and carries forward as intended (e.g. corrections,
  overrides persist and are picked up).
- Establish ONE ground truth (a real computed payslip / finalized row from the
  correct engine path) and diff every surface against it.

## Phase 5 — Regression on neighbours  *(conditional)*

Run only if the change shares data with other screens/flows. Skip with a reason
if fully isolated. Delegate this to `feature-impact-sweep` (path divergence,
denormalized-field drift, cross-screen staleness, duplicates/orphans) rather than
re-deriving it here.

## Phase 6 — Report & verdict  *(always)*  — the shippable artifact

Write a **plain-words, read-only** report (this is the product a buyer sees). It
must contain:
- **Scope** — the change-to-surface map and which personas/phases ran.
- **Coverage** — scenario matrix: each scenario, persona, technique, and
  PASS / FAIL / BLOCKED. State coverage honestly; name anything **not** tested and
  why (no silent gaps).
- **Defect log** — one row per issue: `severity (Blocker/Major/Minor/Cosmetic +
  0–4) | screen:control | steps to reproduce | expected (from oracle) | actual |
  evidence (screenshot / DB row)`. Most severe first.
- **Skipped phases** — each with its one-line reason.
- **Verdict** — **SHIP** or **DON'T-SHIP**, with the must-fix-before-ship list.

Exit criterion for SHIP: no open Blocker or Major. Anything less ships only on an
explicit, recorded waiver.

Save under `tests/j-qa/<feature>-<date>.md`. If the user asked to halt on the
first real defect (standing "halt on finding" rule), file it and stop for a
decision rather than papering over it.

---

## Notes to adapt per codebase

The checklist above generalizes across apps, but a few concrete facts about
*your* system make the difference between a fast, reliable pass and a lot of
false starts. Spend a few minutes filling these in for your own codebase, then
keep this appendix current as a living reference:

- **Browser driving quirks.** Which tool/pane you use, and any known-flaky
  interaction (e.g. a specific dropdown/modal crashing the preview process).
  If `screenshot`-style tools hang, prefer DOM/text-tree reads for assertions
  and push interactive flows to API+DB verification instead of fighting the UI.
- **Route/API layout gotchas.** Any inconsistent prefixing between subsystems
  (e.g. most routes under `/api`, one subsystem at root) that will 404 a route
  that actually exists if you probe the wrong prefix.
- **Local vs LAN networking quirks.** If one host/IP is flaky and another is
  reliable in your dev setup, note which to default to.
- **Frontend API-base-URL config.** If a missing env var makes the frontend
  silently call its own origin instead of the backend (login 404s, blank
  authenticated shell), note the exact fix and where it lives so it isn't
  re-diagnosed from scratch each time.
- **Login timing / race conditions.** If navigating during an in-flight login
  request drops the session, say so, and name the console/network signals that
  distinguish "not logged in" from "app crashed."
- **Dynamic-ref UI quirks.** Any editable grid whose per-cell references shift
  on re-render — verify one input works, then prove the rest at the API/DB
  rung rather than fighting positional UI entry.
- **Auth verification method.** Always verify RBAC through a real login, never
  a header-spoofed request — note the login endpoint/shape and test accounts.
- **New-entity/model runtime registration.** If your ORM/framework requires an
  entity to be registered somewhere beyond its own file (e.g. a DataSource
  `entities:` array) for a real write to succeed, name that requirement — a
  missing registration compiles, migrates, and boots fine, and only 500s on
  the first real write. Run one real write for any feature adding a new
  entity; don't stop at "it boots."
- **Local tooling quirks.** OS/shell-specific gotchas (e.g. temp-path
  resolution issues, a missing CLI tool with a known workaround) worth not
  rediscovering each time.
- **DB access without a GUI/CLI tool.** The fallback method for querying the
  dev DB directly (a scripting language + driver, with sane timeouts), and any
  schema-naming quirk (e.g. per-tenant schemas needing quoting).
- **Build/restart discipline.** The exact sequence after a backend change
  (build → restart → re-hit the real endpoint) — stale build artifacts are the
  first suspect for "the fix didn't work," and a long-running process may
  predate the last rebuild.
- **Where ground truth actually lives.** Name the tables/rows that hold the
  authoritative computed values (not the on-screen preview, which can diverge)
  and the ledger/audit tables for governance actions.
- **Zero/null-result triage.** Before calling an empty or zero result a defect,
  check for a missing fixture precondition (disabled feature, empty config
  table) in the test environment.
- **Ground-truth resolver for your #1 shared quantity.** Name the authoritative
  resolver function and the denormalized copy known to drift from it.
- **Multi-environment machine: confirm the browser is hitting the backend you
  think it is.** On a shared dev machine running multiple branches/worktrees
  against the same database, a stale browser tab or an already-running dev
  server from another lane can silently serve requests through a *different*
  backend build while everything still looks like it's working. Before
  trusting any live-UI result, inspect the actual network request
  origin/port and confirm it matches the backend you started and verified —
  don't assume a config file's declared port is the port actually in use. If
  it's wrong, don't kill someone else's process — start your own frontend on a
  free port pointed at your own backend instead.
