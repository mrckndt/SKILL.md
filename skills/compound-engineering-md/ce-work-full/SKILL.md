---
name: ce-work-full
description: "Self-contained copy-paste execution loop combining proof-first TDD, parallel subagent waves, behavior-preserving simplification, multi-lens code review with an independent validator gate, and fix loop in one file. Use when ce-simplify and ce-code-review are not installed or a single pasteable file is needed."
argument-hint: "[plan path or concrete work description]"
---

# Autonomous Full Execution Loop (TDD + Simplify + Multi-Lens Review + Fix)

Execute the supplied plan or concrete work request end-to-end through one self-contained, disciplined loop:
**Intake -> Proof-First Implementation (Parallel Waves + Cluster Simplify) -> Quality Checks & Simplify -> Multi-Lens Review & Independent Validator -> Fix Loop & Residual Work Gate -> Final Validation**.

Execute all in-scope units continuously in this run; progress updates do not end the run. Stop early only for an explicit blocker. Never estimate human-hours or slice a plan into "this session" subsets (if a plan is genuinely oversized, tell the user plainly to narrow scope before execution).

---

## Core Operating Invariants

1. **Orchestrator owns authority.** Subagents implement bounded units, review diffs, or apply assigned fixes; you inspect actual diffs (`git status`, `git diff`), run authoritative verification, and make every canonical commit.
2. **Fresh worker per unit.** Launch each implementation or review subagent with a fresh context and a bounded packet; retire each worker handle immediately after its unit is integrated.
3. **Preserve user WIP & git safety.** Leave pre-existing user WIP and unpushed commits untouched; stage and commit only explicitly named unit files (`git add <files>` and `git commit -F - -- <files>`); stay off the real default branch unless explicitly directed.
4. **Build what was asked:**
   - **Qualifying safeguards:** Add an unrequested guard, retry, fallback, option, or abstraction ONLY when an existing contract requires it, tracing shows leaving it out lets uncaught harm land (manual human reminders do not count), or adding it later is expensive (stored data/format, public/shared interface, money, security). When uncertain whether a safeguard qualifies, build it. Give passing items their smallest form; report rejected ideas in the summary as considered and not built.
   - **Non-narrowing & conflicts:** Never delay, gate, cap, or skip requested behavior to fit a safeguard. If the plan specified a risky design, build it as specified and report the risk; stop and ask/block only if the conflict was left open.
   - **Non-goals:** Leave plan non-goals unbuilt unless implementation turns up new evidence (then build and report that evidence).
   - **Replacing internal symbols:** When replacing a symbol whose callers are all in-repo, update callers and delete the old version in the same change (unless the plan explicitly retains it).
5. **Two-fix rule.** If two fixes for the same failing check fail, stop patching. Name the shared assumption both fixes relied on, inspect actual code to verify it, and target the root cause. If a flawed assumption came from the plan and correcting it stays within agreed scope, fix it and note what changed; block only if correcting it changes a settled decision, lacks authority, or needs user input.
6. **Respect `session-settled:` decisions.** Preserve user-settled choices without silently working around them. Report real defects inside them at full strength; if implementation proves a settled decision unworkable, stop and report a blocker.
7. **Keep plan IDs out of written files.** Reference `U`/`R`/`A`/`F`/`AE`/`KTD` IDs in summaries and blockers only; inside source code, comments, docs, and scripts, write what the requirement or decision says in plain prose instead of the ID.
8. **Degrade gracefully.** If the harness has no subagent tool, execute the same phases, fresh-lens passes, and gates inline sequentially (disclosing non-independence for review).

---

## Phase 0: Workspace & Plan Intake

1. **Writable checkout & branch safety:**
   Require a writable git checkout (if none is available, stop and report the blocker). Resolve the default branch on its tracking remote (`<remote>/<default>`). If on the default branch, detached `HEAD`, or the default branch is unknown:
   1. If a tracking remote is resolved, run `git fetch -- "<remote>" "<default>"`.
   2. Choose the branch base: `"<remote>/<default>"` when the fetch succeeded and `HEAD` has no commits ahead of it, else `HEAD` (and state that you branched from `HEAD` when the default/remote was unknown).
   3. Create a descriptive feature branch (`git checkout -b "<branch-name>" -- "<base>"`), verify with `git branch --show-current`, and announce it in 1 line.
   Otherwise stay on the invoked feature branch without renaming it.
2. **Pre-work WIP & run-state snapshot:**
   Record and retain these run-state variables across context compaction:
   - `work_base`: `git rev-parse HEAD` immediately after Step 1 branch setup.
   - `last_simplified_sha`: initialized to `work_base`.
   - `pre_work_wip`: `git status --short --untracked-files=all` (extracting repo-relative paths only — destination side on renames — for exclusions) plus pre-existing unpushed commits checked via `git log --oneline "<remote>/<default>..HEAD" --` (if no remote exists OR the command exits non-zero, record `"unknown"` — never treat empty output from a failed probe as "none").
   - `proof_tests`: list of proof test paths recorded during Phase 1.
   Untouched WIP rides along uncommitted without stashing. If an in-scope unit must edit an already-dirty file, ask the user once (covering all colliding files) whether to commit them with the user's edits or leave them uncommitted (recording excluded files in `excluded_wip` as unshipped run-wide exclusions). Never push/publish pre-existing unpushed commits without user authorization.
3. **Plan readiness, intake, or bare-prompt triage:**
   - **Blank or continuation input** (e.g., `"proceed"`): If the conversation holds exactly one current plan or planning brief, execute it; ask if multiple are plausible or none exist.
   - **Plan document:** Verify readiness before executing: code work, scope, and verification must be sufficiently defined with no unresolved `Resolve Before Planning` or launch-blocking `Open Questions` (stop and recommend planning for a requirements-only Product Contract). Verify material existing repo prerequisites before editing (resolve within scope or block without altering the contract).
     - **Multi-PR plans:** If the plan's Execution profile splits units into `### PR<N>` groups, execute ONLY the single requested or earliest unbuilt group whose `Waits on:` prerequisites are satisfied (stop and report `BLOCKED` on unmet `needed before start` or `deploy dependency` waits; never combine separate PR groups onto one branch).
     - **Sizing reads:** For long plans, map headings first (`Goal Capsule`, `Product Contract`, `Planning Contract`, `Implementation Units`, `Verification Contract`, `Definition of Done`), then read `Deferred to Implementation`, `Scope Boundaries` (non-goals), active units, cited `R`/`F`/`AE`/`KTD`, and any Product Contract Key Decision whose `Governs R...` links name those Rs. Leave the plan body unmodified during execution; progress lives in commits and the task tracker.
   - **Bare prompt:** Scan target files and existing tests. **Trivial** (1-2 files, no behavioral change; when uncertain take the fuller route) skips only unit decomposition/task list (keeping test discovery and review if behavior-bearing); **Large** (10+ files, architectural choices, or touches auth/payments/migrations) clarifies key trade-offs before editing.
   - Ask clarifying questions ONLY for genuine ambiguities; execute settled plan scope without renegotiating it.
4. **Derive Implementation Units:** Create dependency-ordered tasks from units with Goal, Files, Dependencies, optional Execution note, and Verification.

---

## Phase 1: Proof-First Implementation Loop

Process units in dependency layers until every in-scope unit is complete.

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
5. **Simplify as you go (cluster boundaries):** Every 2-3 related units or at a natural phase boundary (never after every single unit), when substantive human-authored code changed since `last_simplified_sha` reaches >=30 lines, run Phase 2 Step 3, verify, commit separately, and update `last_simplified_sha = HEAD`.

---

## Phase 2: Core Quality Checks, UI Check & Behavior-Preserving Simplify

1. **Core Checks:** Run the project's full test suite, typecheck, and lint command.
2. **UI Verification (if applicable):** For user-visible UI changes, inspect desktop and mobile widths in a browser when tooling is available (else audit layout code for overflow/responsive breakage and record browser check as unavailable). For Figma-sourced UI when both Figma and browser/screenshot tools are reachable, compare both sides at each design width and re-check after fixes; if either side is unreachable, record which side is unavailable and skip the Figma comparison.
3. **Simplify (when substantive human-authored code changed since `last_simplified_sha` >=30 lines; exclude prose/markdown/docs/configs/generated/lockfiles; skip purely mechanical diffs):**
   Scope to `git diff "<last_simplified_sha>..HEAD" --` (excluding any untouched `pre_work_wip`/`excluded_wip` paths). Dispatch fresh plain **general-purpose** subagents in parallel (full read/shell/git access, no specialist role prompt/schema; queue and wait on capacity limits rather than running inline; fall back to sequential inline passes only when no subagent tool exists or dispatch unrecoverably fails and disclose it; instruct each: *"You report findings; you do not edit files"*; pass scope, `proof_tests`, settled decisions, and governing project/scoped coding standards):
   - **Code Reuse:** Existing project helpers, behavior-equivalent stdlib primitives, or platform/framework guarantees (keep value transformations before projection; never swap serializers/coercions without full value-type verification; newly reachable branches are not dead).
   - **Code Quality:** Redundant/derived state, parameter sprawl, copy-paste drift, leaky abstractions, stringly-typed values where enums/constants exist, 3+ nested conditionals, dead code (verified via configured analysis, else structural search), context-dependent vocabulary, pre-release compatibility scaffolding (only after verifying zero deployed/persisted/public/external/dependent-branch/in-repo callers outside scope), mixed responsibilities added by this diff (balance: preserve comprehension and live abstractions).
   - **Efficiency:** Redundant computations/I/O, N+1 queries, missed safe concurrency, hot-path bloat, recurring no-op updates, TOCTOU existence pre-checks, unbounded memory or missing cleanup.
   - **Test Value** *(dispatch only when scope holds tests; flag ONLY tests added/changed by this diff)*: False-confidence tests, implementation-coupled tests, duplicate coverage, test-only seams introduced by this diff, restated inventories. **Never delete:** failing/unrunnable tests (report as defect), source-inspection tests guarding user-facing keys/bytes/paths, sole tests guarding public/protocol/config/schema/migration/security/release contracts, or protected `proof_tests`. Run any deletion candidate first; recommend deletion only after seeing it pass and naming the remaining covering test (or proving vacuousness).
   - **Comments** *(dispatch only when scope holds comments; flag ONLY comments added/changed by this diff; **delete by default, doubt does not save one**)*: Keep ONLY license headers, external/vendor workarounds still live, public API docs, issue/spec links, tool/compiler/runtime comments (pragmas, build tags, type annotations in comments, doctests, generated-file markers, lint/type suppressions), and standard-mandated comments. For surprises in our code, reshape then delete (if reshape is out of scope, still delete and note the reshape); for unenforced invariants, encode as one in-scope test assertion then delete (keep with a note only if unencodable in scope; never add a runtime check).
4. **Apply Safe Fixes & Commit:**
   - Wait for every simplify subagent to finish. Edit only the resolved diff scope; every code edit preserves exact outputs, errors, side effects, and ordering (when uncertain, skip; comments follow delete-by-default).
   - **Never weaken safety checks** (auth, trust-boundary validation, data-loss protection, security, accessibility).
   - **Protect proof tests:** Never delete, weaken, rename, move, merge, or rewrite any recorded `proof_tests` path (or test-only code it calls). If `proof_tests` is empty/unknown, preserve all tests added or changed by this run. Behavior-preserving edits to production code still apply.
   - **Honor settled decisions** (including `session-settled:` decisions, intentional duplication, or separation).
   - Run **project-wide** typecheck + lint and blast-radius tests; never weaken/delete a test to make checks pass (verify remaining covering tests pass after any test-value removal); make a separate path-limited commit and update `last_simplified_sha = HEAD` before Phase 3.

---

## Phase 3: Multi-Lens Code Review & Independent Validation Gate

Skip ONLY for purely mechanical diffs (formatting, dependency bumps, lint-only fixes, generated files) and record `Code review: skipped (mechanical diff)` with a 1-line reason. **Not mechanical:** single- or multi-file behavior edits, control-flow/error-class/behavior-test changes, or applying external/prior review findings. Never substitute a mental self-review.

### 3a. Prepare Context & Select Read-Only Lenses (Parallel)
- **Diff:** `git diff -U10 --irreversible-delete "<work_base>" --` (no `..HEAD`, excluding untouched `pre_work_wip` paths, so uncommitted run edits stay reviewed). Inventory whole-file deletions (`<deleted-files base="<work_base>">` listing deleted paths, exports, and import-time side effects) and include this inventory in every reviewer prompt.
- **Intent & Policy:** Write a 2-3 line intent summary (+ verbatim user limits). **Exclude all `session-settled:` labels from intent summary and reviewer prompts** (keep them for Stage 3b only). Read base-ref `REVIEW.md` (`git show "<work_base>:REVIEW.md"`) if present, and map applicable standards files (`CODING_STANDARDS.md`, else `CLAUDE.md`/`AGENTS.md`).
- **Select lenses from diff evidence** (announce each conditional lens with a 1-line reason):
  - **Always:** `correctness` (logic, boundaries, null/sentinel reuse, state transitions, lifecycle cleanup, lifecycle-state writers with incomplete guards, error propagation, concurrency/interleaving, deleted files still referenced, stand-in guard fidelity, intent compliance).
  - **Standards:** `project-standards` (when applicable standards files exist or discovery is uncertain; grade each file only against its mapped criteria files, quoting both rule and violating line).
  - **Conditional (spawn ONLY when diff evidence warrants):**
    - `testing` (test/harness files changed, OR meaningful runtime behavior changed with no test work; mutate only an isolated copy if mutation-testing)
    - `security` (auth/authz, public endpoints, user input, feature-flag gates, secrets/logs, deserialization, SSRF/path traversal, crypto, disabled production protections)
    - `reliability` (I/O error handling, timeouts, retry backoff/limits, swallowed errors, resource leaks on error paths, cascading failures, silent-pass CI guards)
    - `performance` (DB/ORM query shape, heavy loops/transforms, batching, caching with material impact)
    - `api-contract` (externally consumed routes, schemas, serializers, published signatures with evidenced downstream callers)
    - `data-migration` (migrations, schema dumps, backfills - NOT model/query-only; pass `<work_base>`) + `deployment-verification` (Go/No-Go checklist, NOT `findings[]`) when migration is risky (destructive DDL, backfills, NOT NULL without default, drops/renames)
    - `maintainability` (large structural refactor or >=200 executable changed lines)
    - `frontend-races` (browser UI async fetches, timers, listeners, lifecycle cleanup)
    - `agent-native` (MCP/agent tools, system prompts, or user capabilities in an agent-surface product)
    - `adversarial` *(consequence-based ONLY)*: explicit request for adversarial review OR a diff-evidenced failure scenario with **material consequences** (unauthorized access, financial loss, persistent data corruption/loss, irreversible external effects, major service disruption, or verification guard concealing one). Name scenario + consequence. Diff size or keywords alone do NOT select it.

**Strict Reviewer Calibration (include verbatim in every reviewer prompt):**
- Fresh read-only plain generic subagent; zero findings is valid. Judge against intended behavior and project rules (Google merge bar).
- **Diff Tiers:** `primary` (changed lines, `pre_existing: false`), `secondary` (surrounding code OR untouched buggy code newly reached/exposed by this diff, `pre_existing: false`), `pre_existing` (unrelated untouched bugs, `pre_existing: true`, graded `50`-`100`).
- **Evidence Tools:** Prefer symbol references > AST > grep; log `callsite completeness: grep-only` in `residual_risks` for exhaustive claims resting only on grep.
- **Suppress (omit from findings, `advisory`, `residual_risks`, and `testing_gaps`):** lint/formatter nitpicks, intentional code, checks already enforced by callers/middleware/framework, restating existing code, generic "consider adding X" without a concrete failure mode, suppressed lint lines (unless violating an explicit rule), subjective complaints without a rule, speculative future concerns.
- **Severity (`P0`-`P3`, independent of confidence):** `P0` (critical breakage, security exploit, data loss), `P1` (high-impact defect, broken contract), `P2` (moderate downside), `P3` (minor).
- **Discrete Confidence Anchors (`0`, `25`, `50`, `75`, `100` ONLY):** `100` (provable from code alone), `75` (traced concrete path with observable impact), `50` (unconfirmed concern; survives primary ONLY if `P0` or soft-bucketed; file critical unconfirmed security at `P0`), `25`/`0` (suppress silently).
- **Quote-the-Line Gate:** Every `75`/`100` finding MUST put `file:line` + verbatim motivating code line(s) in `evidence[0]`; quote both sides for races/swapped args and quote generating constructs for metaclasses/ORMs. Without a quote, step down to `50`.
- **Action-Class & Owner Rubric (use ONLY these exact enum values):**
  - `autofix_class`: `gated_auto` (concrete localized fix) | `manual` (design/cross-cutting fix) | `advisory` (report-only observation).
  - `owner`: `downstream-resolver` (code/test fix) | `human` (product/design/authority decision) | `release` (operational/rollout follow-up).
- **Fields per finding:** `title`, `severity`, `confidence`, `file`, `line`, `evidence` (non-empty array), `why_it_matters` (2-4 sentences leading with observable user/caller impact, then root-cause fix), `suggested_fix` (concrete change grounded in repo patterns; name assumptions if incomplete), `autofix_class`, `owner`, `pre_existing` (bool), `requires_verification` (bool).

Dispatch all selected lenses in parallel, queueing and retrying on capacity backpressure, and collect every terminal outcome (log only non-capacity failed/timed-out lenses in Coverage; if `correctness` or `security` fails, retry once or run it inline).

### 3b. Merge, Filter & Plan Completeness (Orchestrator, in exact order)
1. Merge findings ONLY when they share the same defect and fix path (keep distinct failure modes separate; keep more cautious `autofix_class`). Record multi-lens agreement in `reviewers` as corroboration metadata only (keep confidence unchanged and validate normally).
2. Demote `75`/`100` findings lacking a verbatim quote in `evidence[0]` to `50`.
3. Separate unrelated `pre_existing: true` findings (flip directly depended-on findings to `pre_existing: false` and keep primary).
4. Discard preference-only challenges to `session-settled:` decisions; retain real defects. Apply base-ref `REVIEW.md` overrides/exclusions/caps (count changes).
5. Keep `>=75` and `P0@50` primary (never demote a current `P0`/`P1` solely for being single-reviewer); drop unestablished claims; cap testing-only absence-of-coverage at 1 umbrella per subsystem; route eligible single-reviewer `P2`/`P3` advisories (`testing` -> `testing_gaps`; `maintainability`/`reliability`/`adversarial` -> `residual_risks`); drop remaining `<75`.
6. Check plan requirements completeness when executing a plan (scoped to the active `### PR<N>` group's units and requirements on multi-PR plans): unaddressed plan requirement/unit -> `P1`, `confidence: 100`, `manual` / `downstream-resolver`, `plan_completeness: true` (exempt from diff-only Stage 3c validation).
7. Normalize concrete `P0`/`P1` to `downstream-resolver` unless naming a specific blocking product/authority/release decision. Sort `P0` -> `P3` and assign stable `#` IDs.

### 3c. Independent Validator Gate
Order surviving non-`plan_completeness` `P0`/`P1` (including anchor-50 `P0`s, regardless of owner/class) and actionable findings by severity then `#`, and select up to **8** (expanding past 8 whenever >8 `P0`/`P1` survive so every `P0`/`P1` is validated; skip Stage 3c if 0 findings qualify).

**Pre-step (orchestrator):** Independently classify each selected finding against the 8 **Protected Subjects** BEFORE reading verdicts (classify the actual failure alleged, never keywords alone; validator can add protection, never remove it):
- `memory-safety` (bounds/allocs/null derefs), `concurrency` (locks/races/ordering affecting observable behavior), `data-loss` (destructive writes/deletes/truncation/irreversible migrations), `authorization-authentication` (identity/permissions/ownership/tokens), `injection` (untrusted data altering SQL/commands/templates/paths/markup; text assembly alone is not proof), `public-contract` (evidenced break in externally consumed contract), `secrets-exposure` (hardcoded secrets or secrets/PII in logs/errors/URLs/responses/artifacts), `cryptography` (weak algorithms/password hashes/static keys/IVs/disabled cert checks).

Dispatch **ONE fresh read-only plain generic validator subagent** with the selected findings and the diff:
- Inspect cited files, callers, and guards fresh; evaluate independently; never invent findings; never use lack of disproof as confirmation; never inflate severity/confidence.
- Return ONLY JSON verdicts per `#`: `#`, `status` (`confirmed` | `rejected` | `unresolved`), `protected_subject` (one of the 8 keys or `null`), and 1-sentence evidence-grounded `reason` (noting precondition and whether incidence was measured).
- Outside protected subjects: reject claims not proven by cited code, pre-existing unrelated issues, or cases prevented by surrounding code.
- **Inside protected subjects:** Reject ONLY with specific refuting evidence (quoted guarding code, version-specific docs matching in-force version, commit provenance, or a discriminating test result exercising the trigger). **A general passing suite or assumed framework guarantee is NOT disproof.** Without refuting evidence (or when a file is inaccessible or budget runs out), return `unresolved`.

**Post-Validation Orchestrator Verification & Routing:**
- **Verify protected rejections:** Inspect the cited refutation yourself via read-only tools; reclassify any uncited or unverified protected rejection to **`unresolved`**.
- `confirmed`: Set `validation_status: "confirmed"`; retain existing `autofix_class` and `owner`.
- `rejected` (verified): Drop and count.
- `unresolved` / failure / malformed verdict: **Protected-subject OR `P0`/`P1`** stays primary as a verification gate (`validation_status: "unresolved"`, `autofix_class: "advisory"`, `owner: "human"`, `requires_verification: true`, excluded from `Actionable Findings`); **unprotected `P2`/`P3`** leaves primary findings and is logged in Coverage only.
- **Verdict (over ENTIRE primary set):** `Not ready` (any open `P0` confirmed/unvalidated/`unresolved`, or unaddressed plan requirements), `Ready with fixes` (zero `Not ready` triggers, open `P1` or actionable `P2`), `Ready to merge` (otherwise).

---

## Phase 4: Review Fix Loop & Residual Work Gate

1. **Triage:** Verify each surviving finding with `owner: "downstream-resolver"` AND `autofix_class` in `{"gated_auto", "manual"}` against evidence and agreed scope. **`confidence`, `autofix_class`, and `suggested_fix` do NOT prove benefit or grant permission.** Reject unfounded claims and record why; investigate significant problems lacking a `suggested_fix` before deferring; passing tests do not prove an unverified safety property or authorize changing an agreed requirement. Keep `unresolved`, `advisory`, and `owner: "human"` / `"release"` findings out of automated apply.
2. **Apply Fixes (file-grouped):**
   - Apply small fixes whose code is already in context inline, then **verify and path-limited commit inline fixes first** so any subsequent shared-workspace parallel fix wave starts on a clean committed baseline.
   - Group remaining fixes by `file` (ordered `P0` -> `P3`; keep unrelated files in separate batches; >8 findings on one file -> serial passes by severity) and dispatch parallel worker subagents across disjoint files under Phase 1a's parallel/shared-workspace safety and capacity-backpressure rules. Give every fix worker the governing project and scoped instruction files for its assigned files, user constraints, Phase 1b Worker Guardrails, and its assigned `#` findings. Workers verify evidence still matches, fix only assigned `#` findings, never re-run review, do not commit, and report applied vs. skipped `#` with reasons.
3. **Verify & Commit:** After each parallel fix wave, review diffs (scope = assigned `#` only), run tests (`requires_verification: true` -> at least targeted tests; multi-file -> full suite and lint; use the Two-fix rule if a fix breaks tests), and make a path-limited commit (`fix(review): apply findings #...`).
4. **Loop Bound:** Re-run Phase 3 ONLY if fixes materially altered control flow, contracts, or architecture (max 1 follow-up round; execute one final triage/apply/commit pass on its findings, then proceed unconditionally to Step 5).
5. **Residual Work Gate:**
   - If ANY open `P0`/`P1` in `findings` (confirmed, unvalidated, or `validation_status: "unresolved"`), OR any unresolved problem preventing the requested outcome or invalidating a settled decision, cannot be resolved within scope/permission -> stop and report `BLOCKED` with missing evidence/decision, consequence, and recommendation (group related user decisions).
   - Record non-blocking justified deferrals and non-blocking `unresolved` findings verbatim in the final handoff (and under `## Unapplied review findings` if you later open a PR). Omit skipped low-value nitpicks.

---

## Phase 5: Final Validation & Hand-off

Re-open the original plan/request (to counter context compaction) and re-check active units, requirements, `Verification Contract`, and `Definition of Done` against the diff. **If Simplify or the Review Fix Loop modified any file since the last full-suite run, re-run the full test suite, typecheck, and lint now and record the fresh results.**

Verify before completing:
- [ ] Every in-scope unit, requirement, and `Deferred to Implementation` question is satisfied.
- [ ] Full test suite, typecheck, and lint pass on the final committed tree (exact commands + results recorded).
- [ ] Behavioral changes have test evidence (or an explicitly justified exception).
- [ ] Abandoned/experimental code is removed; pre-existing user WIP and unpushed commits were respected; any excluded dirty files are listed as unshipped.
- [ ] UI/responsive check completed (if applicable).
- [ ] Code review + validator gate completed (`Verdict` + `Coverage`) or explicitly noted as `Code review: skipped (mechanical diff)`.

Present a concise final summary containing:
1. **Completed Work & Commits**
2. **Verification Evidence** (commands run, proof tests, exceptions)
3. **Simplify & Review Outcomes** (`Verdict`, lenses run, findings confirmed/applied/rejected/unresolved, Coverage)
4. **Residual / Deferred Items, Unshipped Exclusions, or Blockers** (if any)
