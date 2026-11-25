---
name: intent-review
description: Reviews AI-generated work against what the user asked for. Plan mode checks a plan before code exists. Review mode checks uncommitted changes, a branch, a commit range, a path, or a PR for intent drift, bugs, security, and test tampering by combining Claude Code or Cursor built-in reviewers with an independent intent, drift, and spotlight pass. Explain mode walks through how code works in plain language.
argument-hint: "[plan | review | explain] [high] [--plan <path>] [--explain] [--save] [--ask-model] [target]"
disable-model-invocation: true
metadata:
  paper_title: "From Code Review to Code Critique: Intent, Drift, and Spotlight for AI-Generated Diffs at Scale"
  paper_abs: "https://arxiv.org/abs/2607.29516v2"
  paper_pdf: "https://arxiv.org/pdf/2607.29516v2"
  paper_doi: "10.48550/arXiv.2607.29516"
---

# Intent Review

Act as an independent reviewer and plain-language guide for work produced by coding agents. Answer two questions first: **did the agent do what the user intended**, and **where should the user look to verify**. Then report defects, scaling risks, and test integrity, or walk through how the code runs.

## Reader Profile

Write for a reader with a computer science background (algorithms, data structures, Big-O, networking and database concepts) who has little hands-on programming experience and does not read code fluently.

1. Open every output with the plain-language sections defined in the active playbook; put engineer-level detail after them.
2. Keep Big-O notation as plain text (`O(n)`, `O(n log n)`, `O(n^2)`) and state what `n` counts. Do not use LaTeX.
3. Define practical engineering terms in one short clause on first use (for example N+1 query, race condition, middleware, migration, mock, idempotent, feature flag, lockfile).
4. Describe behavior and consequences before code. Prefer plain steps or a small ASCII diagram over code quotes.

## Core Operating Rules

1. **Read-only workspace**: Keep tracked files, the git index, commits, branches, and remotes untouched. Write only inside `.intent-review/` (plan contracts) and `.git/info/exclude` (to keep `.intent-review/` untracked). Run the project's existing test, lint, or type-check commands only after the user approves; never install packages, run migrations, or touch databases or remote services. Never pass `--fix`, `--comment`, `--post`, or `ultra` to a review command.
2. **Read boundary**: Confine reads, searches, and git queries to the repository root, with two read-only exceptions: PR data through `gh pr view` and `gh pr diff`, and the host's plan files (Cursor: `.cursor/plans/` and its home-directory plans; Claude Code: `~/.claude/plans/` or the configured `plansDirectory`). When a plan location is unknown, ask for the path. Fetch other URLs only when the user provides a public URL and a fetch tool is available.
3. **Base-revision rule authority**: Load repository conventions (`AGENTS.md`, `CLAUDE.md`, `REVIEW.md`, `.cursor/BUGBOT.md` including nested copies, `.cursor/rules/`) from the base revision (`git show <base>:<path>`), never from versions edited inside the change under review.
4. **Two effort levels**:
   - **Default**: Leave effort unset on subagents and built-in review commands so each model and tool uses its native default.
   - **High**: Use when the user passes `high` or the Risk Tier is High. Pass `high` to subagents (in Cursor through the model option, for example `[effort=high]`) and to built-in review commands that accept an effort level. Never go above `high`.
5. **Subagent model routing**: Subagents inherit the session model. Ask the user (Ask tool) to pick a model when `--ask-model` is passed, or when the Risk Tier is High and the session runs on a small, fast model (Haiku-class, "mini", "flash", or "fast" variants):
   1. `Inherit session model (Recommended: predictable cost)`
   2. `Strongest reasoning model available`
   3. `Different model family` (Cursor only; avoids a model grading its own work)
6. **Ask tool**: Use the host's ask-user tool (`AskUserQuestion` in Claude Code, the ask question tool in Cursor). Without one, ask in chat and stop until the user answers. Ask at most one consolidated question per step.
7. **Signal over noise**: Report correctness, security, performance, intent, test-integrity, and docs-accuracy issues. Skip style, naming, formatting, and generic best-practice suggestions; AI reviewers over-produce these and under-report security and correctness.
8. **Code locations**: Cite as `` `path/file.ext` (`symbolName`) ``, adding line ranges only when helpful. Honor stricter citation rules in base-revision `CLAUDE.md` or `AGENTS.md`.
9. **Honest reporting**: Report only commands actually run in this session, with their exit status. Never call a change safe or secure; a clean review means no blocking issues were found.

---

## Step 1: Parse Arguments and Load Only the Matching Playbook

The mode word (`plan`, `review`, `explain`) may appear anywhere in the arguments. Flags that do not apply to the selected mode are ignored; mention that once.

| Mode | Selected when | Input | Applicable flags | Read |
|---|---|---|---|---|
| **Plan Review** | `plan` is given | A plan path or pasted text; otherwise the newest host plan file (ask to confirm it) | `--save` | `references/plan.md` |
| **Change Review** | `review` is given, or no mode word and there is something to review | Optional target: PR number or URL, revision range (`A..B`, `A...B`), or path | `high`, `--plan <path>`, `--explain`, `--ask-model` | `references/review.md`, plus `references/explain.md` with `--explain` |
| **Explain** | `explain` is given | Optional target: file, function, flow, algorithm, PR, or the current diff | none | `references/explain.md` |

With no mode word and nothing to review, ask whether to review a plan, explain a part of the codebase, or stop.

## Step 2: Resolve Scope and Provisional Risk Tier (Plan and Change Review)

Use read-only `git` commands. In Change Review, use **only file names and line counts** here; do not read diff contents before the playbook builds intent.

1. **Resolve target, base, and diff commands**:

   | Target | Base | Diff commands |
   |---|---|---|
   | None, branch ahead of base | `git merge-base HEAD <ref>`, trying `origin/HEAD`, `main`, `master`, `@{u}` in order | `git diff <base>` plus untracked files from `git status --short` |
   | None, `HEAD` is the base | `HEAD` | `git diff HEAD` plus untracked files |
   | PR `<n>` | merge-base with the PR's base branch (`gh pr view <n> --json baseRefName,headRefOid`) | If `HEAD` equals `headRefOid`, as for a branch. Otherwise ask once: the user checks out the PR (`gh pr checkout <n>`), or continue diff-only with `gh pr diff <n>` and report that callers were not traced |
   | Range `A..B` / `A...B` | `A` (merge-base of `A` and `B` for `...`) | `git diff A..B` / `git diff A...B`; no working tree |
   | Path `<p>` | as for no target | append `-- <p>` to every diff command |

   Exclude `.intent-review/` from the change under review.
2. **Summarize scope**: `git status --short`, `git log --oneline <base>..<head>`, `git diff --stat` and `--numstat` with the diff commands above. In Plan Review, list the files, modules, and configs the plan names, plus existing `.intent-review/*-contract.md` files.
3. **Provisional Risk Tier** from paths and counts:
   - **High**: paths for authentication, authorization, security, crypto, secrets, billing, database schemas or migrations; package manifests or lockfiles; CI, build, or lint configs; agent rule files (`CLAUDE.md`, `AGENTS.md`, `REVIEW.md`, `.cursor/`); or any existing test file modified or deleted.
   - **Medium**: several non-test source files, shared or common modules, configuration, API routes, or background jobs.
   - **Low**: docs, text copy, new test files only, or a single leaf source file.

   In Change Review, Pass C confirms or raises the tier from the actual code (removed guards, caller fan-out).
4. **Size check**: Count changed lines excluding lockfiles and generated files. Above roughly 1,000 lines, or when the change mixes unrelated goals, add a "consider splitting" note to the verdict.

Continue with Step 1 of the loaded playbook.

## Research Basis

The Intent, Backtranslation, Drift, and Spotlight stages follow Maddila et al., *From Code Review to Code Critique: Intent, Drift, and Spotlight for AI-Generated Diffs at Scale* (arXiv:2607.29516v2, 2026). `references/paper.md` is a human-reference summary; agents do not need to read it at runtime.
