---
name: ce-simplify
description: "Simplify code for clarity, reuse, quality, and efficiency while preserving exact behavior, and prune low-value tests and comments. Use standalone on a branch/scope or in pipeline mode after implementation and before code review."
argument-hint: "[blank for current branch diff, or path/scope] [mode:pipeline] [size-cutoff:<N>] [proof-test:<path>] [plan:<path>]"
---

# Behavior-Preserving Code Simplification

Simplify code for clarity, reuse, quality, and efficiency, and prune tests and comments that carry no value, while preserving exact behavior. Prioritize readable, explicit code over compact code; fewer lines is not the goal.

---

## Mode & Scope Resolution

Strip optional tokens `mode:pipeline`, `size-cutoff:<N>`, `proof-test:<path>` (repeatable), and `plan:<path>` before reading the scope. **`proof-test:<path>` and `plan:<path>` are context only, never simplification scope.**

- **Pipeline mode** (`mode:pipeline`, or self-initiated / invoked by another workflow when the user's current message did not ask for it): Work only on the caller scope or current branch diff, apply worthwhile fixes directly without asking, prune only tests/comments added or changed by the current work, return a concise Step 3 summary to the caller (never paste it into a commit, PR body, or other artifact), and continue to the caller's next step without ending the turn.
- **Standalone mode** (user requested directly): Report findings first without editing unless the user explicitly asked to just fix them; requests to `audit`, `find`, `review`, or `list` ALWAYS report first without editing.

Resolve scope in order:
1. Explicit user/caller diff range (e.g., `changes in <sha>..HEAD`, excluding any listed paths) or explicit path/directory scope (authoritative; keep edits strictly within the diff hunks for a diff range, or within the named files/directory for a path scope).
2. Current git branch versus the merge base with the default branch on its tracking remote, including staged and unstaged changes: first compute `BASE=$(git merge-base HEAD -- "<remote>/<default>")` and verify a non-zero-length SHA with exit 0 before running `git diff "$BASE"`. If the default branch/remote or merge-base is unknown or fails, fall back explicitly to `git diff HEAD` and disclose it (never pass an empty substitution into `git diff`).
3. Standalone fallback outside git or without a diff: files edited earlier in this conversation.

If scope resolution finds nothing:
- **Pipeline mode:** Report "nothing to simplify", skip reviewers, and return to the caller's next step (do not end the turn).
- **Standalone mode:** Ask the user what to simplify.

**Preflight:** Code means ONLY human-authored source and tests in a programming language (TS, Python, Go, shell, SQL, etc.). Prose, markdown (including agent skills, prompts, and instructions), docs, plans, data, config, generated/vendored files, dependencies, and lockfiles are NOT code. For mixed scopes, retain only the code and the test fixtures it uses. This check is about kind, never size (explicit small scopes still run). If no code (or only mechanical churn like formatting/dependency bumps/lint-only edits) remains, report "nothing to simplify": return to the caller's next step in pipeline mode, or stop in standalone mode.

---

## Step 1: Prepare Evidence & Launch Parallel Review Lenses

### Dispatch Rules
- **Subagent dispatch:** Dispatch one fresh plain general-purpose subagent per applicable lens in parallel (full read/shell/git access and no specialist role prompt or fixed output schema that overrides the lens rubric). Instruct every subagent: *"You report findings; you do not edit files."*
- **Concurrency & fallback:** On concurrency/capacity limits, queue and wait rather than running inline. Fall back to sequential inline passes only when no subagent tool exists or dispatch unrecoverably fails (disclose the fallback).
- **Targeted standalone focus:** A standalone request about ONLY tests or ONLY comments runs ONLY that reviewer.
- **Single-pass mode (`size-cutoff:<N>`):** Count added/changed lines of non-test, non-blank code (tests remain in scope for review; they only do not count toward the cutoff). At `>= N`, run the full lens team; below `N`, run ONE combined general-purpose subagent seeded with all applicable rubrics verbatim (labeling findings by persona). Report chosen mode and line count in Step 3.
- **Large-scope handling:** For scopes too large for one reviewer to read in full, pass every reviewer an export/signature outline (exports and function signatures without bodies) so each opens only needed files, or split standalone reviews by top-level directory (reporting group totals, strongest findings, and offering the rest on request).
- **Pre-dispatch structural evidence (standalone existing-code scopes only - a path, area, or repo rather than a diff; skip for test-only or comment-only requests):** Gather brief co-change history (`git log` on scope) and import-graph cues (plus the export/signature outline if huge) and pass them to Lens 6.
- **Mandatory payload for every subagent:** Pass (a) its full lens rubric verbatim, (b) the resolved scope (diff, file set, or export outline), (c) mode (`pipeline` or `standalone`), (d) any `proof-test:` paths, user/`plan:` constraints, and project coding standards, and (e) structural evidence for Lens 6.
- **Shared output contract (include in every subagent prompt):** Return each finding with its location (`file` + symbol/line), category/issue, recommended action, and concrete fix or evidence; if nothing qualifies, say so explicitly.

### Lens Rubrics (pass verbatim to each subagent)

#### 1. Code Reuse Reviewer
Find where code duplicates something that already exists, while preserving exact behavior:
- Existing project utilities/helpers that replace new functions or inline logic (name the exact symbol).
- Standard-library or runtime primitives ONLY when behavior-equivalent for the inputs in play (skip swaps with locale, sort-stability, UX, or serialization differences).
- Hand-maintained guarantees already owned by the platform, framework, or downstream contract: name the provider, remove only behavior it directly owns, keep value transformations before downstream projection, do NOT combine with serializer/coercion replacement without tests or direct comparisons covering every relevant value type, and treat newly reachable branches as live (never dead code).

#### 2. Code Quality Reviewer
Find hacky patterns while preserving exact behavior:
- Redundant or derived state, cached values that should be derived, observers/effects that could be direct calls.
- Parameter sprawl instead of generalizing or restructuring existing parameters.
- Copy-paste with slight variation: first check whether an existing source of truth or verified guarantee eliminates the duplication; consolidate only when behavior-preserving; branches made reachable by removing a guard/filter are not dead; replace serializers/coercions only after proving exact equivalence.
- Leaky abstractions or stringly-typed values where constants, enums, or branded types already exist.
- Unnecessary wrapper elements with no layout or behavioral role (component-tree UI frameworks only; skip elsewhere).
- Conditionals nested 3+ levels deep.
- Dead code, unused imports, or unused exports verified via configured project-wide analysis, else structural search (account for re-exports, dynamic imports, and framework conventions; skip if uncertain).
- Context-dependent vocabulary: rename conversation-/iteration-bound or inconsistent terms toward established codebase vocabulary (preserve precise domain terms).
- Pre-release compatibility scaffolding (interfaces/data shapes from an earlier iteration of the unshipped scope) ONLY after verifying no deployed, persisted, public, external, dependent-branch, or in-repo caller exists outside scope and all required caller updates fit inside the mutation boundary (if uncertain, skip).
- Mixed responsibilities (one file holding two unrelated jobs with little shared state and different callers where separating them does not couple them more tightly; file length alone is never a finding). In pipeline mode, flag ONLY mixing the current work introduced or worsened, and move only code the current work added.
- Leave comment findings to the Comment Reviewer.
- *Balance:* Preserve comprehension, named concepts, unrelated logic separation, and abstractions whose testability or extensibility purpose is not verified obsolete.

#### 3. Efficiency Reviewer
Find wasted work and resource problems while preserving exact behavior:
- Redundant computations, repeated file reads, duplicate API/DB calls, N+1 patterns.
- Independent operations run sequentially when they can safely run concurrently.
- New blocking work added to startup or per-request/per-render hot paths.
- Recurring no-op updates: guard polling, event, and reducer updates; verify wrappers preserve the platform's no-change signal (e.g., same-reference return).
- Unnecessary existence pre-checks before operating (TOCTOU anti-pattern) instead of operating and handling errors.
- Memory: unbounded data structures, missing cleanup, listener/handle leaks.
- Overly broad operations (reading entire files or loading all records when filtering for one).

#### 4. Test Value Reviewer *(dispatch only when scope contains tests)*
- **Pipeline scope:** In pipeline mode, flag ONLY tests added or changed by the current work.
- Find tests that cost maintenance without protecting observable behavior:
  - **False-confidence tests:** Expected value computed by the code under test, mock supplies the result the code should produce, negative case rejected by the wrong guard, or vacuous assertions.
  - **Implementation-coupled tests:** Asserts internal mock calls or breaks under behavior-preserving refactors (prefer rewriting at the owning boundary; delete only if a stronger test already covers the behavior).
  - **Duplicate coverage:** Same contract asserted redundantly across layers or near-identical cases belonging in a table-driven test.
  - **Test-only seams:** Production export, flag, wrapper, or hook created solely for tests when the real entry point works (report test + seam together; in pipeline mode, flag only seams introduced by the current work).
  - **Restated inventories:** Copied fixtures, export lists, or source text that merely mirror code.
- **Never recommend deleting:** A failing or unrunnable test (report as possible defect); a source-inspection test that fails when a user-facing key, byte, or path changes and survives an identifier-only rename; the only test guarding a public API, protocol, config/schema, migration/storage format, security, or release/package contract; a protected proof test (or test-only code it calls); or a test solely because it is slow, static, or looks like the implementation.
- **Evidence bar & output fields:** Read the whole test, covered code, callers, overlapping tests, dependency types/source (and git history in standalone mode). Run the test first; recommend deletion ONLY after seeing it pass AND naming the remaining covering test (or proving it asserts nothing). Recommend new tests only as the replacement in a rewrite. Return per finding: location, category, action (`delete`, `merge`, `rewrite`, or `report as defect`), caught failure, remaining covering test (or vacuous-assertion proof), unlocked production seam, and verification method.

#### 5. Comment Reviewer *(dispatch only when scope contains comments)*
- **Pipeline scope:** In pipeline mode, flag ONLY comments added or changed by the current work.
- **Delete by default; doubt does not save a comment** (deleting a comment cannot change behavior, so uncertainty does not justify keeping one). Verify claims like `IMPORTANT`, "do not remove", "too risky", or long justifications against the live code path. Never recommend rewording/shortening in place of deleting.
- **Keep list ONLY:** License/copyright headers; behavior forced by external dependencies/platforms/vendors/protocols still true on a live code path; public API doc comments; links to issues/specs explaining a constraint code cannot express; comments read by tools, compilers, or runtimes (directives, pragmas, build tags, type annotations in comments, doctests, generated-file markers, lint/type suppressions); comments explicitly required by written project coding standards.
- **Outside the keep list:**
  - Restates code, narrates changes, task history, or commented-out code -> delete.
  - Explains a surprise in our own code -> reshape (rename/extract/type) then delete (in pipeline mode, if reshape falls outside scope or changes behavior, still delete the comment and report the reshape as open work).
  - States an unenforced invariant -> encode as a test/type assertion then delete (a comment describing what code already does is not such a rule: delete it; in pipeline mode, encode as one assertion in an in-scope test file or else keep and note why; never add a runtime check just to remove a comment).
- Collapse multiple comments sharing one root cause into a single finding.

#### 6. Structure Reviewer *(dispatch only for standalone existing-code scopes, excluding test-only/comment-only requests)*
Treat the supplied co-change history and import graph as leads to verify in actual code (exempting safety checks):
- Hunt for **hidden links** (co-changing files without an explicit shared contract; also report disagreeing copies with no reachable real input difference as a hidden link, and as a defect only when real input reaches the difference), **needless links** (tight coupling across independent concerns), **surplus** (redundant layers/indirections), and **special cases** (report each special case with the missing requirement it suggests, rather than a local patch).
- Report a structural finding even when removing it changes behavior or a test pins current behavior (state that explicitly so the orchestrator judges applicability after review).
- Report duplicated code, dead code, comments, or tests only when naming the structural link they belong to; propose abstractions only where code varies in more than one place.

---

## Step 2: Collect, Group & Route (Orchestrator)

**Completion barrier:** Wait until every dispatched reviewer reaches a terminal outcome (and release finished review agents when the host requires caller-owned cleanup) before proceeding.

**Cross-reviewer structural grouping (when Lens 6 ran):**
- Group findings across all reviewers under root-cause structural changes ONLY when all three hold: co-changing pieces end up together, independent pieces untie, and dependencies point toward more stable code (include a special case under a structural change only when that change eliminates it; otherwise keep findings separate). Persist groupings to a unique private temp dir created with `mktemp -d "${TMPDIR:-/tmp}/ce-simplify-XXXXXX"` and record its absolute path in run state or the task tracker so it survives compaction.
- **Public-interface contract check:** Verify every exported function/public interface preserves outputs, errors, side effects, and ordering on its own; list changes that cannot as `not applicable` (with reason) and report bug-fixing changes as possible defects.

### Choose Execution Path

- **Path A - Standalone report-first (default for standalone, and mandatory for `audit`/`find`/`review`/`list`):**
  - If nothing qualifies, emit a 1-line "nothing qualifies" message naming the scope and **stop**.
  - Otherwise present: opening line (scope, reviewers run, group totals); continuous numbering across **Delete**, **Refactor** (grouped structural changes numbered with subsumed findings indented unnumbered underneath; `not applicable` last with reason), **Encode as a test**, and a separate list for any pre-existing failing tests (possible defects). Include location, issue, and reviewer evidence per finding (plus all Lens 4 test-deletion fields). End by explaining how the user can apply (all findings, specific numbers, or a described subset), and **stop** (do not edit files or run Step 3).
- **Path B - Standalone follow-up apply (user selects numbers/subset after a report):**
  - Re-check every selected finding against current code first; skip any whose code changed since the report (stating why).
  - Apply remaining selections under the Apply Guardrails below (apply grouped structural changes whole).
  - If a grouped structural change was applied, run Lens 6 once more on the modified files and include its notes without starting another edit round.
  - Proceed to Step 3 (reporting applied vs. skipped by number).
- **Path C - Pipeline mode or standalone "just fix them":**
  - Apply worthwhile findings directly under the Apply Guardrails below; record false positives and low-value findings as skipped without asking.
  - If a standalone grouped structural change was applied, run Lens 6 once more on the modified files and include its notes without another edit round.
  - Proceed to Step 3.

### Apply Guardrails
1. **Mutation boundary:** Inspect outside scope to evaluate findings, but edit ONLY the resolved scope and needed import/export lines. For a user-named file or directory scope, those import/export lines must also sit inside that scope; skip any fix requiring edits outside the boundary.
2. **Exact behavior preservation:** Every code edit must preserve outputs, errors, side effects, and ordering; when uncertain, skip (comment deletions follow Lens 5's delete-by-default rule).
3. **Never simplify away a safety check:** Preserve trust-boundary validation, auth/authz checks, data-loss protection, security checks, and accessibility affordances.
4. **Protect proof tests (pipeline mode):** Protect every `proof-test:` path (else every test named by the session's verification evidence). Skip any finding from ANY reviewer that would delete, weaken, rename, move, merge, or rewrite one, or remove/rename test-only code it calls. Behavior-preserving edits to the production code it exercises still apply. If you cannot identify which tests those are, preserve all tests added or changed by the current work (skip every test deletion and rewrite).
5. **Honor settled decisions:** Preserve settled decisions from any referenced `plan:<path>` or design doc (including `session-settled:` decisions, product constraints, intentional duplication, or deliberate separation).

---

## Step 3: Verify & Summarize

1. Run **project-wide** typecheck and lint. Run tests matched to blast radius (scoped for local changes, broader tests for shared/wide-reach changes, full suite when scoping is unavailable). If no test suite, lint, or typecheck is configured, state that explicitly in the summary rather than silently skipping.
2. Report failures with check name and relevant output. Fix simplification-caused failures or revert the responsible edit. Never weaken or remove a test to make checks pass; remove a test ONLY via a test-value finding and confirm every remaining covering test named by that finding still passes.
3. Summarize: what was already sound, what improved (or state that nothing changed), counts applied by category (`reuse`, `quality`, `efficiency`, `structure`, `tests`, `comments`), skipped count (by number on Path B), single-pass mode/line count if used, check commands and outcomes, and any comment kept because its invariant could not be encoded. Never use net lines removed as the success metric. In pipeline mode, this summary ends the skill, not the turn; continue to the caller's next step.
