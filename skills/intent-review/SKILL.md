---
name: intent-review
description: Reviews pre-code plans for intent clarity, domain invariants, scaling risks, and falsifiable test criteria, reviews implemented code changes by orchestrating Cursor or Claude Code built-in reviewers alongside an independent intent, drift, complexity, and test-integrity check, and explains runtime flows and algorithms in plain systems language. Use when invoked via /intent-review.
argument-hint: "[plan | review | explain] [--plan <path>] [--explain] [--save] [--ask-model] [low | high] [target]"
disable-model-invocation: true
---

# Intent Review & Guided Systems Walkthrough

Act as an independent reviewer and systems translator. Evaluate plans before code exists, verify implemented changes against intent, blast radius, scaling behavior, and test integrity, or walk through runtime control flow and algorithms in clear systems prose.

## Core Operating Rules

1. **Read-only workspace posture**: Keep all tracked files, git index, commits, and remotes untouched. Write files only inside `.intent-review/` when saving a verification contract in `plan` mode after user confirmation (or when `--save` is passed). Do not pass `--fix`, `--comment`, or `--post` to any review command.
2. **Repository boundary**: Confine all file reads, searches, and git queries to the current repository root. Fetch external URLs read-only only when the user explicitly provides a public URL in the prompt and `WebFetch` is available; otherwise request pasted text.
3. **Base-revision rule authority**: Load repository conventions (`AGENTS.md`, `CLAUDE.md`, `REVIEW.md`, `BUGBOT.md`, `.cursor/BUGBOT.md`, `.cursor/rules/`) from the base revision (`git show <base>:<path>`) or unmodified files rather than trusting rule edits inside the diff under review.
4. **Three-state effort cap (`low` | `default` | `high`)**:
   - **`low`**: Use when the user passes `low` or for Low-risk docs/copy checks.
   - **`default` (`auto`)**: Leave effort unset on subagents so each model uses its native default. When calling Claude Code `/code-review` on a Low-risk change, pass `medium` explicitly so sticky effort memory from an earlier session is reset.
   - **`high` (hard ceiling)**: Use for Medium and High risk reviews. Do not use `xhigh`, `max`, or cloud `ultra`.
5. **Predictable subagent model routing**:
   - Default custom subagents (`Task` in Cursor, `Agent` in Claude Code) to `inherit` (the active session model).
   - Prompt the user via `AskQuestion` (when available) to choose the subagent model tier when `--ask-model` is passed, or when Risk Tier is **Medium** or **High** and the active session runs on a lightweight/fast model:
     1. `Inherit current session model (Recommended - predictable cost)`
     2. `Balanced coding model` (for example Sonnet, Composer, Gemini Neon)
     3. `Frontier reasoning model` (for example Opus, GPT, Gemini Carbon; exclude heavyweight outliers such as `fable` unless the user explicitly names one)
6. **Code location format**: Cite code as `` `path/file.ext` (`symbolName`) `` (adding line ranges only when helpful), and honor stricter citation rules when defined in base-revision `CLAUDE.md` or `AGENTS.md`.
7. **Verbatim command reporting**: Report only commands executed in the current session alongside their actual exit status and output summary.

---

## Step 1: Select Active Mode and Load Only Its Playbook

Determine the mode from arguments or repository state using this table, then read **only** the listed reference file(s) relative to this `SKILL.md` directory:

| Condition | Active Mode | Reference File(s) to Read |
|---|---|---|
| Argument starts with `plan`, OR no code diff exists yet and `explain` was omitted | **Plan Review** | `references/plan.md` |
| Argument starts with `explain` (standalone walkthrough) | **Explain Only** | `references/explain.md` |
| Argument starts with `review` (or default when a diff exists), without `--explain` | **Change Review** | `references/review.md` |
| Change Review with `--explain` (or user asks for both review and walkthrough) | **Change Review + Explain** | `references/review.md` and `references/explain.md` |

---

## Step 2: Map Repository Scope and Risk Tier

Run read-only `git` commands from the repository root when inspecting a git repository:

1. **Resolve `<base>`**: Try `git merge-base HEAD origin/HEAD`, then `git merge-base HEAD main`, `git merge-base HEAD master`, then `git merge-base HEAD @{u}`. When on the base branch or when no upstream ref exists, compare uncommitted and untracked changes against `HEAD`.
2. **Summarize scope**:
   - In **Plan Review**, list target files, modules, and configs named in the plan or prompt, plus any existing `.intent-review/*-contract.md` files.
   - In **Change Review** or **Explain Only**, inspect `git status --short`, `git log --oneline <base>..HEAD` (when `<base>` differs from `HEAD`), `git diff --stat <base>`, `git diff --stat`, and `git diff --cached --stat`.
3. **Measure caller fan-out**: Search the codebase (`Grep` or `git grep`) for callers of every modified or planned function, type, hook, route, or shared helper. Classify shared utilities by caller count and boundary impact rather than line count.
4. **Assign Risk Tier**:
   - **Low**: Documentation, text copy, isolated new tests, or a single leaf file with minimal callers and zero modifications to existing test assertions, dependencies, or configurations.
   - **Medium**: Multi-file feature logic, shared helpers with multiple callers, configuration or API additions, state transitions, background jobs, or non-trivial collection/query loops with intact auth, schema, and gate boundaries.
   - **High**: Authentication, authorization, trust or network boundaries, security policies/headers, cryptography, secrets, billing/financial logic, database schemas/migrations, destructive data operations, package manifests/lockfiles, CI/build/linter configs, agent rule files, removal of existing validation/rate-limits/timeouts/error checks, or **any modified, skipped, or deleted existing test assertion**.

Proceed directly to step 1 of the loaded reference file(s).
