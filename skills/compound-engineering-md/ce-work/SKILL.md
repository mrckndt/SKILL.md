---
name: ce-work
description: "Execute a plan or work request end-to-end with proof-first TDD, parallel subagent waves, and caller-owned gates invoking ce-simplify and ce-code-review. Use when ce-simplify and ce-code-review are available as separate skills."
argument-hint: "[plan path or concrete work description]"
---

# Work Execution Orchestrator (Modular)

Execute the supplied plan or concrete work request end-to-end:
**Intake -> Proof-First Implementation (Parallel Waves + Cluster Simplify) -> Quality Checks & `ce-simplify` -> `ce-code-review` -> Fix Loop & Residual Work Gate -> Final Validation**.

Execute all in-scope units continuously in this run; progress updates do not end the run. Stop early only for an explicit blocker. Never estimate human-hours or slice a plan into "this session" subsets (if a plan is genuinely oversized, tell the user plainly to narrow scope in `ce-plan`).

---

## Core Operating Invariants

1. **Orchestrator owns authority.** Subagents implement bounded units or apply assigned review fixes; you inspect actual diffs (`git status`, `git diff`), run authoritative verification, and make every canonical commit.
2. **Fresh worker per unit.** Launch each implementation subagent with a fresh context and a bounded unit packet; retire each worker handle immediately after its unit is integrated.
3. **Preserve user WIP & git safety.** Leave pre-existing user WIP and unpushed commits untouched; stage and commit only explicitly named unit files (`git add <files>` and `git commit -F - -- <files>`); stay off the real default branch unless explicitly directed.
4. **Build what was asked:**
   - **Qualifying safeguards:** Add an unrequested guard, retry, fallback, option, or abstraction ONLY when an existing contract requires it, tracing shows leaving it out lets uncaught harm land (manual human reminders do not count), or adding it later is expensive (stored data/format, public/shared interface, money, security). When uncertain whether a safeguard qualifies, build it. Give passing items their smallest form; report rejected ideas in the summary as considered and not built.
   - **Non-narrowing & conflicts:** Never delay, gate, cap, or skip requested behavior to fit a safeguard. If the plan specified a risky design, build it as specified and report the risk; stop and ask/block only if the conflict was left open.
   - **Non-goals:** Leave plan non-goals unbuilt unless implementation turns up new evidence (then build and report that evidence).
   - **Replacing internal symbols:** When replacing a symbol whose callers are all in-repo, update callers and delete the old version in the same change (unless the plan explicitly retains it).
5. **Two-fix rule.** If two fixes for the same failing check fail, stop patching. Name the shared assumption both fixes relied on, inspect actual code to verify it, and target the root cause. If a flawed assumption came from the plan and correcting it stays within agreed scope, fix it and note what changed; block only if correcting it changes a settled decision, lacks authority, or needs user input.
6. **Respect `session-settled:` decisions.** Preserve user-settled choices without silently working around them. Report real defects inside them at full strength; if implementation proves a settled decision unworkable, stop and report a blocker.
7. **Keep plan IDs out of written files.** Reference `U`/`R`/`A`/`F`/`AE`/`KTD` IDs in summaries and blockers only; inside source code, comments, docs, and scripts, write what the requirement or decision says in plain prose instead of the ID.
8. **Degrade gracefully.** If the harness has no subagent tool, execute the same phases and gates inline sequentially.

---

## Phase 0: Workspace & Plan Intake

1. **Writable checkout & branch safety:**
   Require a writable git checkout (if none is available, stop and report the blocker; if the user requested worktree isolation, invoke `ce-worktree` first — control returns here with the isolated path and branch). Resolve the default branch on its tracking remote (`<remote>/<default>`). If on the default branch, detached `HEAD`, or the default branch is unknown:
   1. If a tracking remote is resolved, run `git fetch -- "<remote>" "<default>"`.
   2. Choose the branch base: `"<remote>/<default>"` when the fetch succeeded and `HEAD` has no commits ahead of it, else `HEAD` (and state that you branched from `HEAD` when the default/remote was unknown).
   3. Create a descriptive feature branch (`git checkout -b "<branch-name>" -- "<base>"`), verify with `git branch --show-current`, and announce it in 1 line.
   Otherwise stay on the invoked feature branch without renaming it.
2. **Pre-work WIP & run-state snapshot:**
   Record and retain these run-state variables across context compaction:
   - `work_base`: `git rev-parse HEAD` immediately after Step 1 branch setup.
   - `last_simplified_sha`: initialized to `work_base`.
   - `pre_work_wip`: `git status --short --untracked-files=all` (extracting repo-relative paths only — destination side on renames — passed as repeated `exclude:<path>` tokens when any path contains a comma, else comma-separated) plus pre-existing unpushed commits checked via `git log --oneline "<remote>/<default>..HEAD" --` (if no remote exists OR the command exits non-zero, record `"unknown"` — never treat empty output from a failed probe as "none").
   - `proof_tests`: list of proof test paths recorded during Phase 1.
   Untouched WIP rides along uncommitted without stashing. If an in-scope unit must edit an already-dirty file, ask the user once (covering all colliding files) whether to commit them with the user's edits or leave them uncommitted (recording excluded files in `excluded_wip` as unshipped run-wide exclusions). Never push/publish pre-existing unpushed commits without user authorization.
3. **Plan readiness, intake, or bare-prompt triage:**
   - **Blank or continuation input** (e.g., `"proceed"`): If the conversation holds exactly one current plan or `ce-plan` brief, execute it; ask if multiple are plausible or none exist.
   - **Plan document:** Verify readiness before executing: code work, scope, and verification must be sufficiently defined with no unresolved `Resolve Before Planning` or launch-blocking `Open Questions` (route a requirements-only Product Contract to `ce-plan`). Verify material existing repo prerequisites before editing (resolve within scope or block without altering the contract).
     - **Multi-PR plans:** If the plan's Execution profile splits units into `### PR<N>` groups, execute ONLY the single requested or earliest unbuilt group whose `Waits on:` prerequisites are satisfied (stop and report `BLOCKED` on unmet `needed before start` or `deploy dependency` waits; never combine separate PR groups onto one branch).
     - **Sizing reads:** For long plans, map headings first (`Goal Capsule`, `Product Contract`, `Planning Contract`, `Implementation Units`, `Verification Contract`, `Definition of Done`), then read `Deferred to Implementation`, `Scope Boundaries` (non-goals), active units, cited `R`/`F`/`AE`/`KTD`, and any Product Contract Key Decision whose `Governs R...` links name those Rs. Leave the plan body unmodified during execution; progress lives in commits and the task tracker.
   - **Bare prompt:** Scan target files and existing tests. **Trivial** (1-2 files, no behavioral change; when uncertain take the fuller route) skips only unit decomposition/task list (keeping test discovery and review if behavior-bearing); **Large** (10+ files, architectural choices, or touches auth/payments/migrations) offers `ce-plan` first unless already sized.
   - Ask clarifying questions ONLY for genuine ambiguities; execute settled plan scope without renegotiating it.
4. **Derive task list:** Create dependency-ordered tasks from units with Goal, Files, Dependencies, Execution note, and Verification.

---

## Phase 1: Proof-First Implementation Loop

Process units in dependency layers until all in-scope units are complete.

### 1a. Scheduling, Dispatch & Parallel Safety Check
- **Inline:** Trivial work, single sequential units chained on their predecessor, or tasks needing mid-flight user input.
- **Serial subagent:** A chained sequential unit when orchestrator context is crowded enough that a fresh window materially helps.
- **Parallel subagent wave (default for independent layers):** Cap at 3-5 workers, dispatched together in one turn, ONLY when all hold:
  1. All dependencies are already committed.
  2. Declared files AND hidden write surfaces (lockfiles, migrations, generated artifacts, shared schemas/types, single dev server/DB/port) are disjoint. Resolve uncertainty by inspecting actual files/contracts rather than blanket serialization; serialize only the specific contending units.
  3. Expected merge/verification cost is smaller than serial authoring, and each unit outweighs worker cold-start ramp-up (batch tiny units together or run inline).
- **Isolation & wave abort:**
  - **Isolated worktree (preferred):** Pass the base commit SHA; worker verifies `HEAD` matches before editing (stopping and falling back to shared-workspace or serial execution on mismatch). Units depending on uncommitted state cannot use isolated worktrees. A worktree worker may commit on its temporary branch only when the harness hands results back as a branch; the orchestrator still owns verification and canonical commits.
  - **Shared workspace (strict contract):** Requires a clean committed baseline, strict exclusive file ownership, **no index-writing git commands** by workers, and **no mutating verification** by workers (at most 1 focused stateless unit test).
  - **Wave abort:** Any unowned write, broad unplanned edit, semantic overlap, or repeated collision aborts the wave and disables further parallel waves for the run. Restore ONLY worker-attributed changes and **preserve any unaccounted delta for user reconciliation**.
- **Dispatch mechanics:** Use a plain **generic** subagent (never a specialist whose own role prompt or tool limits narrow your packet); omit any subagent permission-mode override so the user's configured permission settings apply. On pre-launch dispatch rejection: correct an argument error once, leave capacity-limited work queued, and fall back inline if launch still fails.
- **Worker model & escalate-once:**
  - *Tier selection:* Orchestrator and inline units keep the session model. When the harness exposes a model selector (and session model is above mid-tier), assign the host-advertised balanced mid-tier model (paired with session reasoning effort; never cheapest) ONLY to fully settled units whose Files, Approach, and Test scenarios leave no design choice open. Assign the session model (no override) whenever design choices remain open, a shared contract changes, debugging/investigation is needed, multiple subsystems are touched, or tier is uncertain. Honor explicit user/project overrides.
  - *Refused model:* Retry once with no override if the host refuses a model/effort and note it in the summary.
  - *Escalate once (reasoning only, never environment):* If a mid-tier worker cannot work out the unit or fails code verification (not a missing credential/tool/service), retire it without correction and dispatch one fresh session-model worker starting from its workspace changes (or worktree branch/patch when isolated) + packet + report (allowed to undo within the unit's files).

### 1b. Per-Unit Proof-First Protocol (inline or inside every worker packet)

Give each worker ONLY: Goal Capsule, Definition of Done, its unit section, relevant `Verification Contract` entries, resolved `Deferred to Implementation` answers, cited `R`/`F`/`AE`/`KTD` plus every Product Contract Key Decision whose `Governs R...` links name the unit's cited Rs, `Patterns to follow`, **governing project instructions and any subdirectory-scoped instruction files covering the unit's `Files`**, user constraints (where plan and user limit conflict, **user limit wins** and worker reports the conflict), the **Worker Guardrails** below, and Steps 1-6:

**Worker Guardrails (verbatim in every worker packet):**
- Edit only assigned `Files`; if completion requires unowned files or broader authority, stop without editing them and report the needed scope expansion.
- Make no canonical commits or destructive git commands; on a shared workspace run no index-writing git commands; in an isolated worktree verify `HEAD` matches the passed base SHA before editing (stop on mismatch).
- Follow Core Invariants 4 (`Build what was asked`), 5 (`Two-fix rule`), 6 (`Respect session-settled:`), and 7 (`Keep plan IDs out of written files`, included verbatim).

**Unit Steps:**
1. **Completion signal first:** Check if repo-derived capability is already present and verified on the branch; if so, verify it matches, mark the unit complete, and move on. For out-of-repo deliverables, judge from observed live state, never a clean git tree; re-apply only if safe or authorized.
2. **Test Discovery & Evidence Strategy (before behavior changes):** Locate existing tests covering target files; prefer updating/strengthening an existing test over adding duplicates. Honor any `Execution note`; otherwise default to **proof-first (TDD)** or **characterization-first** for behavior-bearing changes. Skip proof-first only for trivial renames, pure config/styling, or generated/manual-only surfaces (record reason + replacement verification).
3. **Red before Green:** First write/update the smallest focused test proving the current behavior slice and **run it to observe the expected failure**; only then edit production code. Update or remove tests for deleted behavior. Make vague scenarios concrete (exact inputs -> expected outcomes) without inventing unrequested failure handling or building handling just so a scenario can exist. Assert observable boundary behavior (returns, stored state, emitted effects), never internal mock call sequences, values computed by the code under test, or mocks/fixtures supplying the result the code should produce. Use the real entry point rather than adding test-only exports/flags/hooks.
4. **Implement minimal slice:** Implement the minimal slice satisfying the unit's Goal and Files, following existing conventions and `Patterns to follow`.
5. **Green, Multi-Guard & System-Wide Check:** Run targeted tests until green. **Multi-guard proof:** when behavior relies on multiple guards, confirm after implementing that the test fails when each guard alone is removed, restoring each guard afterward. *(In a shared-workspace parallel wave, run at most 1 focused stateless unit test and defer broad tests to orchestrator integration.)* Read actual code (not docs) of callers, callbacks, middleware, hooks up to 2 levels out, and other interfaces exposing the changed behavior, and run impacted tests so nothing working breaks.
6. **Blocked-step protocol & Worker Return Contract:** If the harness refuses an action or an environment dependency is missing, **hold that step and its dependents (never work around it by another route)**, finish independent parts, and report (expected/failing tests are normal work, not blockers). Return: (a) changed file paths, (b) verification evidence (`behavior_changed`, existing tests inspected, tests added/changed, observed red failure or characterization, verification command + result, or exception reason), (c) anything considered and not built, (d) any blocked, refused, or unverified step, and (e) any change outside declared `Files` or needed scope expansion.

### 1c. Orchestrator Integration (after every inline/serial unit and every parallel wave)
1. Inspect the actual tree (`git status`, `git diff`), never trust summaries alone.
2. **Integrate one unit at a time in dependency order:**
   - Inspect actual scope and semantic collisions (re-run colliding units on the advancing base rather than blind-merging).
   - Run authoritative tests. On test failure: recover under the Two-fix rule (or escalate once if mid-tier) before advancing; never commit or start the next layer on a broken tree.
   - Commit the completed unit with path-limited staging (`git add <unit-files>` + `git commit -F - -- <unit-files>`, matching project conventions -> recent `git log` -> `type(scope): description`, avoiding shell expansion).
   - Append its proof-test paths to `proof_tests`, mark the unit completed in the task tracker, and immediately retire its worker handle (when a subagent ran) before revalidating remaining wave results against the advancing tree.
3. Capture verification evidence without fabrication: re-derive only what the tree proves and mark missing fields unverified (never fabricate a red-before-green observation).
4. **Handling blockers:** A held/refused/unverified step keeps that unit open (retry a refused action yourself only within permissions and user limits, never via a disallowed route; commit finished parts only if standalone). Hold blocked units and their dependents, finish any independent unblocked units, then report `BLOCKED`.
5. **Simplify as you go (cluster boundaries):** Every 2-3 related units or at a natural phase boundary (never after every single unit), when substantive human-authored code changed since `last_simplified_sha` reaches >=30 lines, run Phase 2 Step 3 on the diff range `<last_simplified_sha>..HEAD` (excluding `pre_work_wip`/`excluded_wip`), verify, path-limited commit, and update `last_simplified_sha = HEAD`.

---

## Phase 2: Core Quality Checks, UI Check & Simplify

1. **Core Checks:** Run the project's full test suite, typecheck, and lint command on the completed change.
2. **UI Verification (if applicable):** For user-visible UI changes, inspect desktop and mobile widths in a browser when tooling is available (else audit layout code for overflow/responsive breakage and record browser check as unavailable). For Figma-sourced UI when both Figma and browser/screenshot tools are reachable, compare both sides at each design width and re-check after fixes; if either side is unreachable, record which side is unavailable and skip the Figma comparison.
3. **Simplify (conditional):** When substantive human-authored code changed since `last_simplified_sha` reaches >=30 lines (skip purely mechanical diffs), invoke **`ce-simplify`** (`ce-simplify.md`) scoped to the diff range `changes in <last_simplified_sha>..HEAD` (excluding any untouched `pre_work_wip`/`excluded_wip` paths — never bare whole-file names), with `mode:pipeline`, every recorded `proof-test:<path>`, and `plan:<path>` (context only). Whenever `ce-simplify` modifies files, verify, make a separate path-limited commit, and update `last_simplified_sha = HEAD` before Phase 3. If `ce-simplify` is unavailable, do a brief manual pass on that diff while protecting proof tests and safety checks, then verify, commit separately, and update `last_simplified_sha = HEAD`.

---

## Phase 3: Code Review, Fix Loop & Residual Work Gate

**Completion gate:** Code review MUST run before finishing, unless the diff is purely mechanical (formatting, dependency bumps, lint-only fixes, generated artifacts), in which case record `Code review: skipped (mechanical diff)` with a 1-line reason. **Not mechanical:** single- or multi-file behavior edits, control-flow/error-class/behavior-test changes, or applying external/prior review findings. Never substitute a mental self-review or "prior findings already applied".

1. **Invoke Read-Only Review:** Run **`ce-code-review`** (`ce-code-review.md`) with `mode:agent`, `base:<work_base>`, `plan:<path>` (when known), `plan_group:<PR-label>` (when executing a single group of a multi-PR plan), `exclude:<comma-separated pre_work_wip paths>` (when untouched WIP exists), and `depth:full` ONLY when the plan, task, or user explicitly asked for a deep/thorough review. Wait for `status: "complete"`.
   - **If `ce-code-review` fails to load or returns `failed`/`degraded`/`skipped`:** in an interactive session, run a harness-native review if available, fix inline, and record `Code review: harness-native fallback` with a 1-line reason; otherwise record `Code review: skipped (ce-code-review unavailable)` and perform an explicit manual diff scan in Phase 4.
2. **Verify & Triage Findings (caller owns apply, not the reviewer):**
   - From the `mode:agent` JSON, treat `actionable_findings` as the candidate fix queue (valid across `lite`, `focused`, and `full` depths), and read all remaining entries in `findings` (especially `validation_status: "unresolved"` gates and `owner: "human"` / `"release"` items) for the Residual Work Gate without auto-applying them. You may independently inspect an `unresolved` gate: if confirmed in code, fix it like an actionable finding; if refuted with quoted evidence, close it; otherwise hold it for Step 5.
   - Verify each candidate against evidence and agreed scope. **`confidence`, `autofix_class`, and `suggested_fix` do NOT prove benefit or grant permission.** Reject unfounded claims and record why; investigate significant problems lacking a `suggested_fix` before deferring; passing tests do NOT prove an unverified safety property or authorize changing an agreed requirement. Defer items needing a user product choice or outside scope/permission.
3. **Apply Fixes (file-grouped):**
   - Apply small fixes whose code is already in context inline, then **verify and path-limited commit inline fixes first** so any subsequent shared-workspace parallel fix wave starts on a clean committed baseline.
   - Group remaining fixes by `file` (ordered `P0` -> `P3`; keep unrelated files in separate batches; >8 findings on one file -> serial passes by severity) and dispatch parallel worker subagents across disjoint files under Phase 1a's parallel/shared-workspace safety and capacity-backpressure rules. Give every fix worker the governing project and scoped instruction files for its assigned files, user constraints, Phase 1b Worker Guardrails, and its assigned `#` findings. Workers verify evidence still matches, edit only assigned `#` findings, never re-run review, do not commit, and report applied vs. skipped `#` with reasons.
4. **Verify & Commit Fixes:** After each parallel fix wave, review diffs (scope = assigned `#` only), run tests (`requires_verification: true` -> at least targeted tests; multi-file -> full suite and lint; use the Two-fix rule if a fix breaks tests), and make a path-limited commit (`fix(review): apply findings #...`). Re-run `ce-code-review` ONLY if fixes materially altered control flow, contracts, or architecture (max 1 follow-up round; execute one final triage/apply/commit pass on its findings, then proceed unconditionally to Step 5).
5. **Residual Work Gate:**
   - If ANY open `P0`/`P1` in `findings` (whether validated, `lite`/`focused`, or `validation_status: "unresolved"`), OR any unresolved problem preventing the requested outcome or invalidating a settled decision, cannot be resolved within scope/permission -> stop and report `BLOCKED` with missing evidence/decision, consequence, and recommendation (group related user decisions).
   - Record non-blocking justified deferrals and non-blocking `unresolved` findings verbatim in the final handoff (and under `## Unapplied review findings` if you later open a PR). Omit skipped low-value nitpicks.

---

## Phase 4: Final Validation & Hand-off

Re-open the original plan/request (to counter context compaction) and re-check active units, requirements, `Verification Contract`, and `Definition of Done` against the diff. **If Simplify or the Review Fix Loop modified any file since the last full-suite run, re-run the full test suite, typecheck, and lint now and record the fresh results.**

Verify before completing:
- [ ] Every in-scope unit, requirement, and `Deferred to Implementation` question is satisfied.
- [ ] Full test suite, typecheck, and lint pass on the final committed tree (exact commands + results recorded).
- [ ] Behavioral changes have test evidence (or an explicitly justified exception).
- [ ] Abandoned/experimental code is removed; pre-existing user WIP and unpushed commits were respected; any excluded dirty files are listed as unshipped.
- [ ] UI/responsive check completed (if applicable).
- [ ] Code review completed (`Verdict` + `Coverage` summary) or explicitly noted with its exact skip/fallback phrase (plus manual diff scan if review was unavailable).

Present a concise final report with:
1. **Completed Work & Commits**
2. **Verification Evidence** (commands run, proof tests, exceptions)
3. **Simplify & Review Summary** (`Verdict`, lenses run, findings applied/rejected/unresolved)
4. **Residual / Deferred Items, Unshipped Exclusions, or Blockers** (if any)

**Optional next steps (offer cleanly without auto-pushing):**
- If this run corrected a wrong belief about how the codebase behaves (a failed approach, real review defect, or wrong assumption/doc), suggest `ce-compound` (or invoke `ce-compound mode:non-interactive` in non-interactive runs; **control returns** here on completion) before shipping so the lesson travels with the PR.
- To push and open a pull request, transfer control to `ce-commit-push-pr` passing repeated `exclude:<path>` tokens for pre-work paths not committed this run plus any `## Unapplied review findings`; if pre-existing unpushed commits exist OR their status is `"unknown"`, transfer to `ce-commit` instead so unoffered commits are not published. For browser QA before shipping UI changes, suggest `ce-dogfood` or `ce-test-browser`.
