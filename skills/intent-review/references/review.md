# Change Review Playbook (`review` Mode)

Review implemented changes in two views:

- **Forest**: what the change was meant to do (Intent), what it actually does (Backtranslation), and the gap between them (Drift).
- **Trees**: verified defects and the few code regions most worth a human check (Spotlight), plus test and guard integrity.

Contents: Step 1 Intent · Step 2 Review passes · Step 3 Drift · Step 4 Verify and rank · Step 5 Evidence check · Step 6 Output

Work best in a fresh session, not the one that wrote the code. When this session wrote the code, say so under Verdict, because its view of the code can leak into intent.

## Step 1: Establish Intent (Forest)

Build intent **without reading the diff**, so the comparison in Step 3 is not circular.

1. **Collect intent sources**, in this order:
   - `--plan <path>`, or a `.intent-review/*-contract.md` whose `Target files` overlap the changed files (or whose `Branch` matches the current branch when not on `main`, `master`, or `trunk`), plus its `Source plan` files. Do not trust a contract added or modified inside the diff under review unless the user confirms it, and ask before reading any `Source plan` file outside the repository root unless passed via `--plan`.
   - The host's plan files for this work (see Core Operating Rule 2; confirm before reading outside the repository root).
   - What the user asked for in this session: user messages and agent replies, excluding tool calls and tool output.
   - Commit messages, and the PR title and body.
   - Issues, specs, or docs inside the repository that those sources link to.
2. **Write the Goal List**: 2 to 8 numbered goals, each an imperative sentence describing an outcome ("Lock an account after 5 failed logins"), plus any stated out-of-scope items and the user's focus area.
3. **Ask only when needed**: When no source states the goal (common in a fresh session without a plan), ask **one** question for the original prompt or plan path, the focus area, and out-of-scope items; include the model choice when Core Operating Rule 5 requires it. When goals come from sources other than the user's own words, proceed and mark them `inferred` in the output.

## Step 2: Run the Review Passes

Every pass starts from a fresh context without this conversation's history. Scale the work to the provisional Risk Tier:

| Tier | Pass A (built-in) | Pass B | Pass C | Effort |
|---|---|---|---|---|
| Low | bug reviewer | yes | short (`3, 7, 8` if no source files changed; `1, 3, 4, 7, 8` if any source file changed) | Default |
| Medium | bug reviewer | yes | full | Default |
| High | bug reviewer + security reviewer | yes | full | High (Pass B stays Default) |

`high` from the user raises the effort, not the passes. Launch the Pass B and Pass C subagents together in one message (background `Agent` calls in Claude Code, parallel subagent calls in Cursor), then run Pass A, and wait for all of them. If Pass C raises the tier to High, run the security reviewer before Step 4.

### Pass A: Built-in Reviewers

**Claude Code**:
1. Run `/code-review [high] <target>` (report only). Pass `high` only at High effort; without a level, `/code-review` reuses the level the user last typed, which counts as the user's own default. Target by case:
   - no target and `HEAD` is the base: omit the target (it reviews uncommitted changes);
   - branch ahead of base: `<base>...HEAD`;
   - PR: the PR number; range: the range as given; path: the path.

   A revision-range target may leave out uncommitted edits; Pass C always covers them. `/code-review` reads `CLAUDE.md` but not `REVIEW.md`, so Pass C applies `REVIEW.md` rules.
2. Security reviewer (High tier, or when the user asks for security review): if `git rev-parse --verify origin/HEAD` succeeds and the target is the current branch, run `/security-review`. Its prompt excludes denial of service, rate limiting, resource exhaustion, and on-disk secrets; Pass C checks those.

**Cursor** (Cursor 3.7+; invoke via `Task`, or `/review-bugbot` and `/review-security` which wrap them):
1. Bug reviewer: call `Task` (`subagent_type: "bugbot"`, `description: "Bugbot"`, `run_in_background: false`) with prompt `Full Repository Path: <absolute repo path>\nDiff: <"branch changes" | "uncommitted changes">` (use `"uncommitted changes"` when reviewing uncommitted work on the base branch, otherwise `"branch changes"`; add `Custom Instructions: <focus>` when provided). Bugbot reads `.cursor/BUGBOT.md` files, not `.cursor/rules/`.
2. Security reviewer (High tier, Pass C tier raise to High, or user-requested security review, on checked-out branch or uncommitted-changes targets—do not gate on `origin/HEAD` or being ahead of base when `Diff` is `"uncommitted changes"`): call `Task` (`subagent_type: "security-review"`, `description: "Security Review"`, `run_in_background: false`) with the same prompt shape.

**Fallback**: Use the prompt below in one read-only subagent when a built-in reviewer is unavailable, blocked, or requires something missing (for example Cloud Agents for Cursor security review); when it reports the diff exceeds its size limit or skips generated files central to the change; or, in Cursor, when the scope is a revision range, a non-default base, a path subset, or a PR that is not checked out. Whenever High tier, a Pass C tier raise to High, or a user-requested security review skips a built-in security reviewer—including missing `origin/HEAD` in Claude Code, blocked or unavailable reviewers, or PR diff-only, revision-range, non-default-base, or path-scoped targets—you **must** run this Pass A Fallback subagent with `"Focus on security"` added to the focus line. In Cursor, mention that the user can also run `/agent-review` with Deep depth for a second built-in opinion.

```text
Perform a read-only bug, regression, edge-case, and security audit. Keep all files and git state untouched and stay inside <absolute repo path>.

Scope:
- Base ref: <base>
- Diff commands: <exact read-only commands>
- Changed files: <list>
- Base-revision project rules: <summary>
- User focus area: <focus or "None">

1. Read the full diff (and the callers and callees of each changed function when the target revision is available locally).
2. Find concrete logic errors, broken edge cases, race conditions, missing error handling, null/boundary failures, and security issues (auth bypass, injection, SSRF, XSS, path traversal, secret exposure, missing rate limits, resource exhaustion). Skip style and best-practice suggestions.
3. Return only high-confidence findings: Severity (Critical|High|Medium|Low), New or Pre-existing, Location (`path/file.ext` (`symbolName`)), exact code quote, trigger scenario, suggested fix. If clean, say "No Pass A issues found."
```

When the change edits rule files that built-in reviewers read (`CLAUDE.md`, `.cursor/BUGBOT.md`), those reviewers follow the edited rules. Say so under Verdict and rely on Pass C for rule checks.

### Pass B: Blind Backtranslation

Spawn one read-only subagent at default effort. Give it **only** the prompt below: no goals, plan, contract, commit messages, or PR text, because those would bias the backtranslation.

```text
Describe what a code change does, using only the code. Keep all files and git state untouched and stay inside <absolute repo path>. Do not read commit messages, PR descriptions, plan files, or .intent-review/.

Scope:
- Base ref: <base>
- Diff commands: <exact commands covering committed, staged, unstaged, and untracked changes, or the PR diff command>

1. Read the diff (`git diff <base>` or PR diff) and untracked text files, including manifests, configs, scripts, tests, and docs. Read surrounding context in existing files only when needed to understand a hunk, and use `git diff --stat` and diff headers for lockfiles, generated files, and deleted files instead of reading whole or deleted files on disk.
2. Write a numbered Task List: one line per task the change performs, phrased as the instruction someone would have been given to produce it ("Add a 5-second timeout to the payment API call"). List dependency, config, test, and doc changes as their own tasks. Do not judge quality.
3. Write a Change Structure paragraph: 3 to 5 plain sentences describing the change as a whole, for a reader who does not read code.
```

### Pass C: Integrity and Defect Reviewer

Spawn one read-only subagent with this prompt. For a Low tier, include only tasks 3, 7, and 8 when no source files changed (docs, text copy, or new test files only), or tasks 1, 3, 4, 7, and 8 when any source file changed so Pass C can confirm or raise Provisional Low and trigger security review before Step 4.

```text
Perform an independent read-only review of implemented changes. Keep all files and git state untouched and stay inside <absolute repo path>.

Context:
- Base ref: <base>
- Diff commands: <exact commands covering committed, staged, unstaged, and untracked changes, or the PR diff command>
- Base-revision rules (including REVIEW.md if present): <summary>
- Risk tier definitions: <copy from SKILL.md Step 2>; provisional tier: <tier>
- Goal List: <numbered goals, marked inferred where applicable>
- Out of scope: <items or "None stated">
- User focus area: <focus or "General review">

Tasks:
1. Blast radius and tier: Find callers of each modified function, type, route, or shared helper (list at most 10 per symbol). Confirm the provisional tier or raise it, with the reason. A tier is raised by removed guards, changed shared helpers with many callers, or touched trust boundaries.
2. Prior-commit guarantees: Pick up to 5 modified or deleted existing blocks, prioritizing guards, tests, and shared helpers. For each, run `git log -n 3 -L <start>,<end>:<file> <base>` (or `git blame` + `git show`) and flag any earlier fix, edge case, or invariant this change weakens or undoes.
3. Tests and gates: Flag deleted tests, newly skipped tests, weakened assertions, tests rewritten to mock away the real unit, and edits to CI, lint, or agent rule files. Say whether a Goal requires each removal (cite it). For each new test, name one realistic one-line production bug that would make it fail; flag tests that would pass anyway.
4. Removed guards and scaling: Flag removed auth or permission checks, tenant scoping, input validation, rate limits, timeouts, resource caps, error handling, and logs. Trace loops, recursion, and I/O on changed paths; state the scaling (for example O(n) over <what n counts>, O(n^2), N+1 queries) and flag superlinear or unbounded paths on variable-size input.
5. Fit: Check sibling modules and handlers. Flag code that ignores an existing safe helper, diverges from sibling error or guard handling, copies a deprecated pattern, or is much more complex than needed.
6. Rollback and diagnostics: Check that behavior is unchanged when a new flag or setting is unset, that migrations or state changes can be reversed, and that failures produce actionable errors or logs.
7. Dependencies and docs: For each new dependency, confirm it is declared in the manifest and lockfile and flag names that look like typos or near-copies of popular packages. For changed docs, READMEs, or comments, verify each factual claim against the code.
8. Out-of-scope violations: Flag changes to anything listed as out of scope.

Skip style, naming, and generic best-practice suggestions. Return numbered candidate findings: Severity (Critical|High|Medium|Low), Category (correctness|security|performance|intent|test-integrity|docs), New or Pre-existing, Location, exact snippet, impact, suggested fix, and how a non-programmer could check it (behavior to try, not code to read). Also return the confirmed tier and one short paragraph per task stating what you checked.
```

## Step 3: Score Drift

Compare the Goal List with the Pass B Task List. Cite goal and task numbers for every judgment.

1. **Goals**: mark each `Met`, `Partial`, or `Not met`. A `Partial` or `Not met` goal is a **DNF** (did not follow).
2. **Unrequested changes (UC)**: every task that traces to no goal is a UC. Classify it `Beneficial` (for example a missing null check or added tests for the requested change), `Neutral`, or `Harmful` (risky or clearly beyond scope: touches auth, data, dependencies, or configs; weakens, skips, or deletes tests, or alters unrelated tests; deletes functionality; breaks callers; or adds substantial unrelated code). Only Harmful UCs count against alignment.
3. **Drift label** (informational; the decision in Step 6 uses goals, UCs, and findings, because the paper shows the middle labels are unreliable):
   - **Perfect alignment**: all goals Met, no Harmful UCs.
   - **Minor drift**: all goals Met, only trivial deviations.
   - **Moderate drift**: most goals Met but some Partial, and/or noticeable Harmful UCs.
   - **Significant drift**: only some goals Met, and/or substantial Harmful UCs that add risk.
   - **Major drift**: no goals Met, near-complete deviation, and/or severe Harmful UCs such as deleted functionality or breaking changes.

## Step 4: Verify, Deduplicate, and Rank Findings

Check every candidate from Pass A and Pass C yourself:

1. **Location check**: Inspect each cited location—on disk for working-tree and checked-out-branch reviews, via `git show <head-of-range>:<path>` for revision ranges (`A..B`, `A...B`), and via `gh pr diff <n>` for diff-only PR reviews. When a finding has no quote, locate the code it describes. Drop a finding only when the location does not exist in that target or the code contradicts the claim.
2. **Critic check**: Keep a finding only if it passes all five:
   - **Correct**: the issue is reachable and not disproved by surrounding callers, guards, types, or sanitizers.
   - **Relevant**: it is a real defect or a gap against the Goal List.
   - **Consistent**: it agrees with the base-revision project rules.
   - **Actionable**: it has a concrete fix.
   - **Accepted**: a senior engineer on this codebase would agree it is worth changing.
3. **Rank**: Merge duplicates, label each finding `New` or `Pre-existing`, assign Confidence (High|Medium|Low), and sort by severity. The top 5 New findings form the **Spotlight**.

## Step 5: Evidence Check

1. **Unbacked claims**: When the session, commit messages, or PR body claim that tests or checks pass, look for the command and its output in this session. Report claims without that evidence as unverified.
2. **Optional run**: When the tier is Medium or High and the repository defines a test, lint, or type-check command (in a manifest, Makefile, README, `CLAUDE.md`, or `AGENTS.md`), ask once whether to run it. Run only approved, existing commands and report the exact command and exit status. For a Low tier, skip this and mention the command under "Try it yourself".

## Step 6: Emit the Review

Keep everything above "Engineer details" under about 40 lines. Include "Engineer details" only when the decision is not "No blocking issues", or when the user asks for it.

````markdown
## In plain words
<Pass B Change Structure paragraph, corrected where Step 4 found it wrong. Name unrequested additions explicitly.>

## Did it do what you asked?
*Goals from: <sources>; marked (inferred) where you did not state them.*
- ✓ | ◐ | ✗ <goal> — <one-line evidence>
- + <unrequested change> — Beneficial | Neutral | Harmful (<why>)

**Drift**: Perfect alignment | Minor | Moderate | Significant | Major drift — <deciding reason in one sentence>

## Verdict
- **Decision**: No blocking issues | Check before merging | Fix first
- **Why**: <one or two sentences>
- **Risk tier**: Low | Medium | High (<what was touched>)
- **Passes run**: <commands and subagents, with models>
- **Checks run**: `<command>` → <exit status and result> | None (static review only)
- **Notes**: <only when they apply: unverified "tests pass" claims; edited rule files steering built-in reviewers; reviewed in the session that wrote the code; PR reviewed diff-only; consider splitting>

## Spotlight: where to look
*(Write "Nothing needs a closer look." when empty.)*
1. **[Severity] [Category] <Title>** (Confidence: High|Medium|Low) — `path/file.ext` (`symbolName`)
   - **What can go wrong**: <concrete consequence in plain words>
   - **How to check**: <behavior to try or question to ask the agent, without reading code>
   - **Fix**: <direct instruction>

## What to do next
- **Prompt for your coding agent** (include when the decision is not "No blocking issues"):
  ```text
  <3 to 8 lines: fix each Spotlight finding, complete each Partial or Not met goal, remove each Harmful UC, keep out-of-scope items untouched, and add or restore tests that fail without the fix. Ask it to show the test command and its output.>
  ```
- **Try it yourself**: <2 to 4 concrete checks of behavior: what to run or click, what input to use, what result to expect>

## Engineer details
- **Blast radius**: <callers, routes, workflows, adjacent features>
- **Scaling**: <for example O(n) over <what n counts>; flag O(n^2), N+1, unbounded memory>
- **Fit and simpler alternative**: Fits cleanly | <concrete alternative>
- **Rollback, defaults, diagnostics**: <behavior when unset, reversibility, error and log visibility>
- **Earlier guarantees**: Intact | <weakened fix or invariant>
- **Tests and gates**: <modified, skipped, or deleted tests; strength of new tests>
- **Removed guards, dependencies, docs accuracy**: None | <details>
- **Pre-existing issues**: None | <brief list>
- **Open questions**: <1 to 3 decisions only a human can make, when uncertainty remains>
````

**Decision rules**:
- **Fix first**: any verified Critical or High New finding; any `Not met` goal; any Harmful UC in a High-risk area; or any weakened, skipped, or deleted test that no goal requires.
- **Check before merging**: any other Spotlight finding; any `Partial` goal or other Harmful UC; or any note about unverified test claims or edited rule files.
- **No blocking issues**: otherwise, at any Risk Tier. Add: "An AI review can miss bugs; this is not a security sign-off."

When `--explain` was passed, append the walkthrough from `references/explain.md` for the changed execution paths.
