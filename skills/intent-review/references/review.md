# Change Review Playbook (`review` Mode)

Review implemented changes (branch commits, staged/unstaged edits, commit ranges, paths, or PR targets) by combining built-in bug/security reviewers with an independent assurance pass over intent drift, prior-commit guarantees, algorithmic complexity, and test integrity.

## Step 1: Establish Intent, Plan Contract, and Model Choice

1. **Locate saved contracts or plan files**:
   - Check `--plan <path>` first. If omitted, check `.intent-review/*-contract.md` for a contract matching the current branch or touched area.
   - If a contract lists `Source Plan / Prompt` paths on disk, read both the contract and those source plan files read-only.
2. **Confirm missing context in one step**:
   - Check whether **Original intent** (goal, spec, issue, prompt, or saved contract) is available. Gather intent from explicit user text, saved contracts, or linked plans rather than inferring intent solely from the diff.
   - Check whether model selection requires confirmation under Core Operating Rule 5 (`--ask-model`, or Medium/High Risk on a lightweight session model).
   - If either item needs user input, ask **one** combined question (using `AskQuestion` when available) covering: (a) Original intent + any specific area of concern ("what you want scrutinized most") + out-of-scope items, and (b) subagent model choice when triggered.

## Step 2: Run Pass A and Pass B in Parallel

Spawn fresh review passes without the builder's conversation history so each reviewer evaluates repository state independently. Run **Pass A** and **Pass B** concurrently in the same turn. Select effort using Core Operating Rule 4 (`low` when requested; native `default`/`auto` for Low risk, passing `medium` only to Claude Code `/code-review`; `high` for Medium and High risk).

### Pass A: Built-in Bug and Security Reviewers (with Fallback)

Select the active host environment from this table:

- **In Cursor**:
  1. Match `Diff` to the review scope: use `"uncommitted changes"` when reviewing uncommitted work on the base branch (or when requested); use `"branch changes"` for branch-against-base reviews.
  2. When the target is `"branch changes"` or `"uncommitted changes"` on the default base branch, launch `bugbot` (`subagent_type: "bugbot"`, `description: "Bugbot"`, `run_in_background: false`, fresh single-shot call) with this exact prompt:
     ```text
     Full Repository Path: <absolute repo path>
     Diff: <"branch changes" | "uncommitted changes">
     ```
     Append `Custom Instructions: <user focus>` only when the user stated specific review instructions.
  3. When Risk Tier is **High**, security review was requested, or auth/trust/network/crypto/input/config boundaries changed, also launch `security-review` (`subagent_type: "security-review"`, `description: "Security Review"`, `run_in_background: false`, fresh single-shot call) with the same `Full Repository Path` and `Diff` block.
  4. **Trigger Pass A Fallback** whenever `bugbot` or `security-review` is unavailable (enum/plan restriction or worker environment), reports that the diff exceeded its size limit, skips generated files that are central to the change, or when the user requested a custom commit range, non-default base branch, or path subset.

- **In Claude Code**:
  1. Invoke `/code-review <medium|high> [target]` in report-only mode (`medium` for Low risk, `high` for Medium/High risk; omit `--fix`, `--comment`, `--post`, and `ultra`). Note that local `/code-review` reads `CLAUDE.md` and skips `REVIEW.md`, so include any base-revision `REVIEW.md` rules in Pass B.
  2. When Risk Tier is **High**, security review was requested, or auth/trust/network/crypto/input/config boundaries changed, check `git rev-parse --verify origin/HEAD`. When `origin/HEAD` resolves and the target is the current branch, invoke `/security-review`. Note that `/security-review` skips denial-of-service, rate-limiting, resource exhaustion, and on-disk secrets; Pass B verifies those areas explicitly.
  3. Wait for background review subagents to finish before starting Step 3.
  4. **Trigger Pass A Fallback** whenever `/code-review` or `/security-review` is blocked (such as `user-invocable-only` overrides or missing `origin/HEAD` on a High-risk change).

- **Pass A Fallback Prompt** (spawn one fresh read-only `generalPurpose` subagent in Cursor or `Agent` in Claude Code, applying the chosen subagent `model` and effort cap):
  ```text
  Perform a read-only bug, regression, edge-case, and security audit. Keep all files and git state untouched, and stay strictly inside <absolute repo path>.

  Review scope:
  - Base ref: <base>
  - Diff command: <exact read-only git diff command>
  - Changed files: <list from git diff --stat / git status>
  - Base-branch project rules (CLAUDE.md / AGENTS.md / REVIEW.md / BUGBOT.md / .cursor/rules): <summary of rules loaded from base ref>
  - User focus area: <focus or "None specified">

  Tasks:
  1. Read the full target diff and surrounding callers/callees in each touched file.
  2. Identify concrete logic errors, broken edge cases, race/concurrency hazards, missing error handling, null/nil/boundary failures, and security vulnerabilities (auth bypass, injection, SSRF/egress, XSS, path traversal, secrets exposure, missing rate limits, resource exhaustion).
  3. Return only high-confidence findings with: Severity (Critical|High|Medium|Low), New vs Pre-existing, File & Symbol (`path/file.ext` (`symbolName`)), Exact code quote, Trigger scenario, and Suggested fix. If clean, state "No Pass A issues found."
  ```

### Pass B: Independent Intent, Drift, Complexity & Test-Integrity Reviewer

Spawn one fresh read-only subagent (`subagent_type: "generalPurpose"` in Cursor, or `Agent` in Claude Code, applying the chosen subagent `model` and effort cap) with this exact self-contained handoff prompt:

```text
Perform an independent read-only assurance review of implemented changes. Keep all files and git state untouched, and confine all reads and git queries to <absolute repo path>.

Context packet:
- Base ref: <base>
- Diff commands to inspect: <exact git diff / git status commands covering committed, staged, unstaged, and untracked files>
- Base-branch rules (loaded from <base>, including REVIEW.md and BUGBOT.md if present): <summary>
- Stated intent: <user's stated goal>
- User focus area: <user's concern or "General review">
- Out-of-scope boundaries: <stated exclusions or "None stated">
- Saved contract / plan files: <paths in .intent-review/ or --plan, or "None">

Execute these 6 tasks in exact order (complete Task 1 from the code BEFORE reading Stated intent into your comparison):

1. Blind Diff Backtranslation:
   - Inspect every added, modified, deleted, and untracked file (including manifests, lockfiles, configs, and scripts).
   - Write a 2 to 4 sentence factual summary of what runtime behavior, data flow, and system state actually changed based strictly on the code.

2. Intent & Plan-Contract Drift Check:
   - Compare your Task 1 summary against Stated intent, Out-of-scope boundaries, and any Saved contract / plan files.
   - Map every requirement and contract item to `Met`, `Partially met`, or `Not met`, citing both production code (`path/file.ext` (`symbolName`)) and test assertions, with Confidence (`High`, `Medium`, `Low`).
   - Flag unrequested behavior changes, silent plan deviations, unrelated refactors, or unexpected dependency additions.

3. Prior-Commit Intent Check (Existing Code Defense):
   - For every modified or deleted existing production block or existing test, run `git log -n 3 -L <start>,<end>:<file>` (or `git blame` + `git show`) against `<base>` to check why those lines were introduced.
   - Flag any prior feature guarantee, edge-case fix, or invariant that this change weakens or undoes.

4. Test & Gate Monotonicity Check:
   - Inspect all touched test files, CI workflows, linter configs, and agent instruction files.
   - Flag any deleted test, newly skipped test (`.skip`, `t.Skip`, `xit`, `@pytest.mark.skip`), weakened assertion/matcher, or test rewritten to mock away the real unit under test.
   - Counterfactual check: for each new test, state one concrete single-line production bug that would make that test fail. Flag tautological tests that only mirror mock wiring or implementation structure.

5. Algorithmic Scaling, Removed Guards & Pattern Hygiene:
   - Trace loops, recursion, collection passes, lock scope, and I/O calls across changed paths and their callers. State time/memory/I/O scaling where non-trivial (for example O(1), O(n), O(n^2), O(n^3), N+1 DB/network calls in a loop, or unbounded memory growth) and flag superlinear or unbounded paths on variable-sized inputs.
   - Flag silent removals of guards (auth/permission checks, tenant scoping, input validation, rate limits, timeouts, resource caps, error handling, diagnostic logs).
   - Check neighboring modules and sibling endpoints/handlers to verify whether added code reused current safe helpers, matched sibling error/guard/idempotency handling, and steered clear of deprecated or incomplete patterns; check whether a much simpler design achieves the same goal.

6. Rollback, Safe Defaults & Diagnostics:
   - Check whether runtime behavior stays unchanged when any new flag/config is unset or disabled (when applicable), whether migrations/state changes roll back safely, and whether failure paths emit actionable errors or logs.

Return a structured report with one section per task (1 through 6) plus a numbered list of candidate findings (`Severity`, `New | Pre-existing`, `Location`, `Exact snippet`, `Impact`, `Suggested fix`).
```

## Step 3: Two-Stage Verification, Deduplication, and Ranking

Audit all candidate findings from Pass A and Pass B yourself before writing the output:

1. **Stage 1 - Deterministic anchor check**: Open each cited file on disk and confirm that the file, symbol, and quoted code snippet exist verbatim in the current tree. Drop any hallucinated path or mismatched quote.
2. **Stage 2 - Semantic reachability check**: Read surrounding callers, guards, types, and sanitizers to confirm the issue is reachable and real. Drop false positives disproved by surrounding code.
3. **Deduplicate and rank**: Merge overlapping findings across Pass A and Pass B. Label each surviving issue `New` (introduced by this change) or `Pre-existing` (already present in untouched logic), assign Confidence (`High`, `Medium`, `Low`), and sort by severity (`Critical`, `High`, `Medium`, `Low`).

## Step 4: Emit the Change Review Decision Brief

Output the review using this structure:

```markdown
## Verdict
- **Decision**: SAFE TO MERGE | NEEDS FIXES | NEEDS HUMAN SIGN-OFF
- **Risk Tier**: Low | Medium | High (<boundaries touched and caller fan-out>)
- **Human Spotlight**: <1 to 3 highest-consequence judgments on intent fit, blast radius, or scaling that deserve human attention, or "None - routine low-risk change">
- **Review passes run**: <exact passes and subagent models/commands executed>
- **Checks/tests executed in this session**: Yes (`<command>` -> `<exit code and result>`) | Static review only (no test commands executed in this session)

## What Actually Changed, Complexity & Blast Radius
- **Observed behavior change (from code)**: <3 to 6 plain-language bullets from blind backtranslation>
- **Plan / Intent drift**: Matches intent and plan | <exact skipped items, unrequested changes, or architecture shifts>
- **Blast radius & callers affected**: <callers, routes, workflows, or adjacent features sharing the touched code>
- **Algorithmic & I/O scaling**: <practical scaling of changed paths, e.g., O(n) single pass, or flag O(n^2)/O(n^3)/N+1/memory risks>
- **Architectural fit / simpler alternative**: Fits cleanly | <concrete simpler or safer approach>
- **Rollback, safe defaults & diagnostics**: <behavior when unset/off if applicable, migration reversibility, and error/log visibility>

## Requirement & Contract Coverage
| Requirement / Contract Check | Status (`Met` \| `Partial` \| `Not met`) | Production & Test Evidence (`path/file.ext` (`symbolName`)) | Confidence |
|---|---|---|---|

**Out of scope / deferred**: <items left untouched, or None>

## Test, History & Control Integrity
- **Prior-commit guarantees ("Feature A vs. Feature B")**: Intact | <prior commit intent weakened or broken>
- **Existing tests or gates modified/deleted**: None | <exact assertion/gate change and whether protection dropped>
- **Counterfactual strength of new tests**: Strong (fails on real behavior regression) | Partial | Weak/Tautological (<reason>)
- **Removed guards or copied-pattern risks**: None | <removed checks, timeouts, rate limits, or flawed copied patterns>

## Verified Findings (Highest Severity First)
*(Write `No verified defects found.` when clean.)*

1. **[Critical|High|Medium|Low] [New|Pre-existing] <Concise title>** (Confidence: High|Medium|Low)
   - **Impact**: <concrete effect on behavior, security, data, scaling, or operations>
   - **Location**: `path/file.ext` (`symbolName`)
   - **Suggested fix**: <direct, actionable instruction ready for an engineer or coding agent>

## Open Questions & Decisions to Escalate (Include When Medium/High Risk or Uncertainty Remains)
- <1 to 3 targeted architectural, domain, security, migration, or rollout questions naming the exact uncertainty>
```

When `--explain` was also requested, read `references/explain.md` (if not yet loaded) and append the **Guided Logic & Flow Walkthrough** for the changed execution paths.
