# Plan Review Playbook (`plan` Mode)

Check a plan, spec, issue, or prompt **before** implementation: is the intent clear enough that an agent cannot misread it, does it fit the real codebase, and does it define tests that an agent cannot pass by cheating.

## Step 1: Locate the Plan and Establish Intent

1. **Locate the plan**: Use, in order, a path or pasted text given after `plan`; the plan in this conversation; or the newest file in the host's plan folders (see Core Operating Rule 2; confirm with the user before reading any plan outside the repository root, and confirm the selected host plan is the right one). When none exists, ask for a path or pasted text. Read plan files read-only.
2. **Extract** from the plan and the user's messages:
   - **Goal List**: 2 to 8 numbered goals, each an imperative sentence describing an outcome.
   - **Focus area**: the decision or risk the user is least sure about.
   - **Out of scope**: adjacent behavior or files the plan intentionally leaves alone.
3. **Ask only when needed**: When the goals are missing or ambiguous, ask **one** consolidated question (Ask tool) before judging the plan. Otherwise proceed and state the defaults you assumed.
4. **Scope and Provisional Risk Tier**: Apply `SKILL.md` Step 2.2–2.3 to list the plan's named files, modules, and configs (plus existing `.intent-review/*-contract.md` files) and assign the Provisional Risk Tier.

## Step 2: Ground the Plan in the Codebase

Check the repository rather than trusting the plan text:

1. **Targets and callers**: Read every file, interface, route, or schema the plan edits or imitates. Find existing callers and consumers that will feel the change.
2. **Domain invariants**: Read base-revision `CLAUDE.md`, `AGENTS.md`, `REVIEW.md`, `.cursor/BUGBOT.md`, `.cursor/rules/`, and nearby docs and tests. Record rules the plan must keep (tenant scoping, idempotency, permission checks, transaction order, backwards compatibility).
3. **Simpler alternative**: Look for an existing helper, config hook, middleware, or data structure that solves part of the problem with a smaller blast radius.
4. **Scaling**: Estimate data volume and call frequency on the planned path. Flag nested scans (O(n^2), O(n^3)), per-item database or network calls in loops (N+1 queries), wide locks, or unbounded memory, and propose a lower-complexity design. Say what `n` counts.
5. **Rollout and diagnostics**: Check behavior when a new setting is unset or off, whether data or schema changes roll back cleanly, and which errors or logs make failures visible.
6. **Falsifiable checks**: Write concrete positive, negative, and boundary cases, including cases proving that forbidden actions stay blocked and adjacent features keep working, so an implementing agent cannot pass with happy-path or tautological tests.

## Step 3: Emit the Plan Review

````markdown
## In plain words
<3 to 5 sentences: what the plan will build, how it fits the existing system, and the single most important thing to settle before building.>

## Verdict
- **Readiness**: Ready to build | Revise plan first
- **Risk tier**: Low | Medium | High (<boundaries touched, caller fan-out>)
- **Decide before building**: <1 or 2 decisions that need the user's judgment>

## Goals and boundaries
1. <goal>
- **Ambiguities an agent would guess at**: None | <list>
- **Out of scope**: <files, workflows, behaviors to leave untouched>
- **Your focus area**: <assessment, or "General pass">

## Fit with the codebase
- **Rules to preserve**: <invariants found, or "None documented; consider recording them">
- **Simpler alternative**: Plan fits cleanly | <alternative grounded in existing files>
- **Affected callers**: <shared functions, schemas, routes, workflows>
- **Scaling**: <for example O(n) single pass over <what n counts>, or the O(n^2) / N+1 risk and a fix>
- **Rollout and diagnostics**: <behavior when off, migration safety, error visibility>

## Checks the implementation must pass
- **Must work**: <input → expected result>
- **Must reject or preserve**: <blocked, malformed, boundary, and regression cases>

## Prompt for your next session
```text
<3 to 5 lines.
- Revise plan first: tell the planning session to resolve the listed ambiguities, lock the out-of-scope items, and add the checks above, using its own plan format.
- Ready to build: tell the implementing session to implement <plan source> while honoring the goals, boundaries, rules, scaling limits, and checks in .intent-review/<slug>-contract.md (when saved).>
```
````

## Step 4: Save the Contract

Leave external plan files untouched so tool-specific plan formats stay valid.

1. When `--save` was passed, save immediately. Otherwise ask (Ask tool) whether to save `.intent-review/<slug>-contract.md` for the implementation and the later `/intent-review review`, or keep the review in chat.
2. Before the first save, resolve the local exclude file via `git rev-parse --git-path info/exclude` (skip if not in a git repository) and add `.intent-review/` if it is not already ignored, so the contract never shows up as a change under review.
3. Write:

```markdown
# Contract: <Short Title>

- **Date**: <YYYY-MM-DD>
- **Branch**: <current branch, or the planned branch name>
- **Target files**: <paths or globs named in the plan>
- **Source plan**: <plan file path(s) or "Chat prompt">
- **Risk tier**: <Low | Medium | High>

## Goals
1. <goal>

## Out of Scope
- <item>

## Rules and Scaling Limits
- <invariant or complexity/query bound>

## Checks
- [ ] <must work>
- [ ] <must reject or preserve>
```
