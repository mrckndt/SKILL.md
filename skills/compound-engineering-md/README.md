# compound-engineering-md

Portable, script-free Markdown skills distilled from [Compound Engineering](https://github.com/EveryInc/compound-engineering-plugin). Every file is self-contained, requires zero bundled scripts or plugin config, and can be pasted into any coding agent or installed as agent skills.

## Core Loop

```text
[ce-strategy] -> [ce-ideate]
                     |
                     v
   +-----> ce-brainstorm    (define WHAT: Product Contract)
   |            |
   |            v
   |       ce-plan          (define HOW: research, units, deepening, doc review)
   |            |
   |            v
   |       ce-work          (proof-first TDD -> ce-simplify -> ce-code-review)
   |            |
   |            v
   +------ ce-compound      (hold the lesson: test/lint -> CODING_STANDARDS -> docs/solutions/)
```

| File | Role |
|---|---|
| `ce-brainstorm/SKILL.md` | Define **WHAT** to build through structured dialogue (`Product Contract`) |
| `ce-plan/SKILL.md` | Plan **HOW** to build and verify it (research, units, deepening, doc review) |
| `ce-work/SKILL.md` | Modular execution loop (proof-first TDD, parallel waves, invokes `ce-simplify` + `ce-code-review`) |
| `ce-simplify/SKILL.md` | Behavior-preserving code, test, and comment simplification (standalone or pipeline) |
| `ce-code-review/SKILL.md` | Read-only multi-lens code review with an independent validator gate |
| `ce-compound/SKILL.md` | Capture lessons in their cheapest durable form (check, standard, `docs/solutions/`) or `refresh` stale learnings |
| `ce-work-full/SKILL.md` | Self-contained single-file execution loop (`ce-work` + `ce-simplify` + `ce-code-review`, copy-paste ready) |

## Upstream Strategy, Ideation & Decisions

| File | Role |
|---|---|
| `ce-strategy/SKILL.md` | Author/update repo-root `STRATEGY.md` via pushback interview + stress test |
| `ce-ideate/SKILL.md` | Six-frame repo-grounded divergent ideation + fresh-context adversarial filter (`docs/ideation/`) |
| `ce-pov/SKILL.md` | Decisive, project-grounded verdict (`Adopt`/`Trial`/`Hold`/`Reject`/`Not-our-problem`), doc take, or approach position |
| `ce-bakeoff/SKILL.md` | Blind 3-baker competing solution development + fresh `ce-pov` judge + counterexample verification |
| `ce-doc-review/SKILL.md` | Standalone multi-lens review of requirements, plans, or specs with interactive walkthrough |
| `ce-prototype/SKILL.md` | Throwaway visual/interactive prototype + `decisions.md` + Product Contract write-back |

## Debugging & Understanding

| File | Role |
|---|---|
| `ce-debug/SKILL.md` | Root-cause diagnosis and test-first fix loop (causal-chain gate, dirty-tree check, post-fix `ce-simplify` + `ce-code-review`) |
| `ce-explain/SKILL.md` | Evidence-backed explanation of how/why code works, diff walkthroughs, or work recaps |
| `wtf/SKILL.md` | Plain-language explanation of the last message, file, link, or passage with all caveats intact *(manual invoke only)* |

## Quality, Browser QA, UX & Optimization

| File | Role |
|---|---|
| `ce-standards/SKILL.md` | Draft or refresh `CODING_STANDARDS.md` & `REVIEW.md` from PR history, fix commits, and `docs/solutions/` |
| `ce-dogfood/SKILL.md` | Autonomous diff-scoped browser QA: Mermaid flows, test matrix, small auto-fixes, persona paper cuts (`docs/dogfood-reports/`) *(manual invoke only)* |
| `ce-test-browser/SKILL.md` | Focused end-to-end browser checks on routes touched by the current branch or PR |
| `ce-polish/SKILL.md` | User-directed live-browser UX polish loop with hot reload and local commit close *(manual invoke only)* |
| `ce-optimize/SKILL.md` | Measured optimization loop (`type: hard` or `type: judge`) with attribution, worktree experiments, and holdout confirmation |

## Git & Pull Request Workflow

| File | Role |
|---|---|
| `ce-worktree/SKILL.md` | Safe git worktree isolation (detects existing isolation, prefers native tools, avoids git footguns) |
| `ce-commit/SKILL.md` | Convention-aware local commits with explicit path-limited staging and outcome-first subjects |
| `ce-commit-push-pr/SKILL.md` | Commit, push, and open/update GitHub PRs with decision-cost-sized, outcome-first descriptions |
| `ce-resolve-pr-feedback/SKILL.md` | Centrally judge PR comments on merit, fix valid items, push back with evidence, reply & resolve threads |

## Writing, Continuity & Environment

| File | Role |
|---|---|
| `ce-noslop/SKILL.md` | Author, edit, or detect AI writing tells while keeping every source fact, identifier, and caveat |
| `ce-handoff/SKILL.md` | Pointer-first session handoff (`create` / `resume`) across context resets without premature actions |
| `ce-retro/SKILL.md` | Evidence-gated session retrospective fixing repo environment friction and routing code/standard work to owners |

## Shared Repo Artifacts

| Artifact | Produced / Maintained By | Consumed By |
|---|---|---|
| `STRATEGY.md` | `ce-strategy` | `ce-ideate`, `ce-brainstorm`, `ce-plan`, `ce-pov`, `ce-dogfood` |
| `CONCEPTS.md` | `ce-brainstorm`, `ce-compound` | `ce-ideate`, `ce-brainstorm`, `ce-plan`, `ce-strategy`, `ce-explain` |
| `CODING_STANDARDS.md`, `REVIEW.md` | `ce-standards`, `ce-compound` | `ce-code-review`, `ce-retro` |
| `docs/plans/` | `ce-brainstorm`, `ce-plan`, `ce-prototype` | `ce-doc-review`, `ce-work`, `ce-code-review`, `ce-explain`, `ce-commit-push-pr` |
| `docs/solutions/` | `ce-compound` | `ce-plan`, `ce-debug`, `ce-explain`, `ce-pov`, `ce-ideate`, `ce-standards` |
| `docs/ideation/` | `ce-ideate` | `ce-brainstorm` |
| `docs/dogfood-reports/` | `ce-dogfood` | Resumed `ce-dogfood` runs & PR review |
| `docs/optimize/` | `ce-optimize` | PR evidence & follow-up `ce-compound` runs |

## Usage

- **Modular workflow (installed skills):** `ce-brainstorm` -> `ce-plan` -> `ce-work` -> `ce-compound` -> `ce-commit-push-pr`.
- **Copy-paste workflow:** Paste `ce-work-full/SKILL.md` with a plan path or task description into any model.
- **Standalone / vibe coding:** Run `ce-debug`, `ce-simplify`, `ce-code-review`, `ce-optimize`, `ce-explain`, or `ce-noslop` directly on any branch or task, and `ce-dogfood` / `ce-polish` on any feature branch.

## License

MIT. Derived from the Compound Engineering Plugin (Copyright (c) 2025 Every).
