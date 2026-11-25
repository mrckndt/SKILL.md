# Plan Review Playbook (`plan` Mode)

Evaluate a proposed plan, spec document, issue description, or prompt **before** implementation begins by checking intent clarity, domain invariants, caller blast radius, algorithmic scaling, and falsifiable verification criteria against the real codebase.

## Step 1: Confirm Intent, Focus Area, and Boundaries

Check whether the prompt or plan document already states:
1. **Core goal**: Desired user/system behavior and problem solved.
2. **Review focus**: Specific architectural choice, migration, edge case, or tradeoff the author is least sure about.
3. **Out-of-scope boundaries**: Adjacent behaviors or files intentionally excluded.

When Core goal is missing or ambiguous, ask **one** consolidated question (using `AskQuestion` when available) to capture the missing items before judging the plan. When Core goal is clear, proceed directly and note defaults for any unstated items.

## Step 2: Ground the Plan Against the Real Codebase

Inspect the repository directly rather than trusting assumptions inside the plan text:

1. **Target files and callers**: Read every file, interface, route, or schema the plan proposes to edit or mirror. Use `Grep` to find existing callers and state consumers that will experience the change.
2. **Domain invariants**: Read `CLAUDE.md`, `AGENTS.md`, `REVIEW.md`, `BUGBOT.md`, `.cursor/rules/`, and nearby module docs/tests for touched directories. Record existing domain rules (such as tenant scoping, idempotency, permission checks, transaction ordering, or backwards compatibility) that the plan must preserve.
3. **Simpler alternatives**: Check whether an existing helper, configuration hook, middleware, or data structure in the codebase already solves part of the problem with smaller blast radius.
4. **Algorithmic and I/O scaling**: Estimate data volume and call frequency on the proposed path. Flag designs that introduce nested collection scans (\(O(n^2)\) or \(O(n^3)\)), per-item database/network requests inside loops (N+1 queries), wide lock contention, or unbounded in-memory growth, and state a concrete lower-complexity design.
5. **Safe rollout, defaults, and diagnostics**: Verify what happens when a new setting or feature is unset/disabled (when applicable), whether data/schema changes can roll back cleanly, and which errors, return statuses, or logs make failures observable at runtime.
6. **Falsifiable verification scenarios**: Write concrete positive, negative, and boundary test cases (proving forbidden actions stay blocked and existing adjacent behavior stays intact) so an implementing agent cannot pass with tautological happy-path tests.

## Step 3: Emit the Plan Review Brief

Output the review using this structure:

````markdown
## Plan Verdict
- **Readiness**: READY TO BUILD | REVISE PLAN FIRST
- **Estimated Risk Tier**: Low | Medium | High (<boundaries touched and caller fan-out>)
- **Human Spotlight**: <1 to 2 highest-consequence architectural, scaling, or boundary decisions that need human judgment>

## Intent, Focus & Boundaries
- **Core goal understood as**: <2 to 3 plain sentences>
- **Requested focus check**: <assessment of the user's named concern, or "General architecture & risk pass">
- **Ambiguities or missing decisions**: None | <specific gaps an implementing agent would otherwise guess at>
- **Locked out-of-scope boundaries**: <files, workflows, or behaviors to leave untouched>

## Codebase Fit, Complexity & Invariants
- **Domain invariants to preserve**: <rules found in base repo docs/code/tests, or "None documented - consider recording key invariants">
- **Simpler or lower-blast-radius alternative**: Current plan fits cleanly | <concrete alternative grounded in existing repo files>
- **Blast radius & callers affected**: <shared functions, schemas, routes, or workflows>
- **Algorithmic & scaling assessment**: <expected time/memory/query scaling such as O(n) single pass vs. O(n^2) or N+1 risk>
- **Rollback, safe defaults & runtime diagnostics**: <behavior when unset/off if applicable, migration safety, and error visibility>

## Falsifiable Verification Criteria (Pre-Code Contract)
- **Must pass (intended behavior)**: <concrete input -> expected state/output>
- **Must reject / preserve (negative & regression cases)**: <concrete blocked, malformed, boundary, or adjacent-feature cases that prove existing behavior survives>

## Ready-to-Paste Handoff Prompt
```text
<Provide a 3 to 5 line prompt tailored to Readiness:
- When REVISE PLAN FIRST: instruct the user's planning tool or session to update the plan using its own format rules to resolve the listed ambiguities, lock the out-of-scope boundaries, and add the falsifiable verification criteria (referencing .intent-review/<slug>-contract.md if saved).
- When READY TO BUILD: instruct the implementing session to implement <plan-source> while honoring all boundaries, domain invariants, scaling constraints, and verification criteria in .intent-review/<slug>-contract.md.>
```
````

## Step 4: Save the Verification Contract (Cross-Session Handoff)

Keep external plan files untouched so tool-specific formats (such as multi-file spec directories or managed plan frontmatter) remain valid:

1. When `--save` was passed, write `.intent-review/<slug>-contract.md` immediately. Otherwise, ask via `AskQuestion` (when available) whether to save `.intent-review/<slug>-contract.md` for the upcoming implementation and `/intent-review review` sessions or keep the brief in chat only.
2. When saving `.intent-review/<slug>-contract.md`, write this compact contract:

```markdown
# Verification Contract: <Short Title>

- **Date**: <YYYY-MM-DD>
- **Source Plan / Prompt**: <plan file path(s) or "Chat prompt">
- **Estimated Risk Tier**: <Low | Medium | High>

## Core Intent
<2 to 3 sentences>

## Locked Out-of-Scope Boundaries
- <item>

## Domain Invariants & Scaling Constraints
- <invariant or complexity/query bound>

## Falsifiable Acceptance & Negative Checks
- [ ] <positive check>
- [ ] <negative / boundary / regression check>
```
