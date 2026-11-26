---
name: ce-commit-push-pr
description: "Commit, push, and open or update a GitHub pull request with an outcome-first description, or draft/rewrite a PR description alone. Use when shipping a branch to a PR or writing a PR body."
argument-hint: "[PR ref] [mode:pipeline] [description-only] [exclude:<comma-separated paths>]"
---

# Commit, Push, and Pull Request

Ship working changes to a GitHub pull request with a crisp, reviewer-centered PR description — or compose/rewrite a PR description on its own.

## Modes

- **Description-only** (`description-only` token OR user asks only to write/draft/describe a PR body): Run **Step 3** only and print the title & body; apply only if asked.
- **Description update** (refresh/rewrite an existing PR's description with no commit/push): Resolve target & verify an open PR exists (exit-0 `[]` = none, stop; non-zero = unknown, resolve auth/connectivity and stop), run **Step 3** in PR mode, preview/confirm (apply directly in `mode:pipeline`), and apply via `gh pr edit` (skipping `gh pr edit` if the new body matches the existing body).
- **Full workflow (default):** Run Steps 1 -> 2 -> 3 -> 4.
- **`mode:pipeline`:** Non-interactive caller mode — ask zero blocking prompts using conservative defaults (keep existing PR body untouched unless in description-update mode, stop as a blocker if target/base/push remote cannot be resolved or if pre-existing unpushed commits exist).

---

## Step 1: Resolve Target, Remotes, Branch & Existing PR State

Run each git/gh probe as its own call and interpret exit codes directly (re-verify right before pushing and right before `gh pr create`):
1. `git rev-parse --show-toplevel` (stop if not a git repo), `git status --short --untracked-files=all`, `git branch --show-current`, `git log --oneline -10`.
2. **Resolve targets once (passed to every later git/gh call):**
   - Resolve the target PR repository `(host, base_owner/base_repo)` from the PR URL or checkout remotes (`gh repo view --json parent,nameWithOwner,url` — for fork checkouts, target the parent/base repo). Pass `-R "<host>/<base_owner>/<base_repo>"` explicitly to EVERY `gh` call below.
   - Resolve `<base-remote>` (the local remote matching `<base_owner>/<base_repo>`), `<push-remote>` (the branch's configured push/upstream remote, else the fork/user remote), and `<default>` (the default branch on `<base-remote>`). If `<base-remote>`, `<push-remote>`, or `<default>` cannot be positively resolved (**unknown**): stop and report the blocker (never guess a literal remote or branch name).
3. **Branch safety:**
   - **Detached `HEAD`:** Create a descriptive feature branch (`git checkout --no-overwrite-ignore -b "<branch-name>"`) and re-read `git branch --show-current`.
   - **On `<default>` with work to do:** Fetch `git fetch -- "<base-remote>" "<default>"`, then check `git log --oneline "<base-remote>/<default>..HEAD" --` (non-zero exit = **unknown** -> stop). If unpushed local commits exist: interactive -> ask (<=2 options) whether to carry them onto the new branch or branch from `"<base-remote>/<default>"`; `mode:pipeline` -> stop and report a blocker so unoffered commits are not silently published. Create the branch with `git checkout --no-overwrite-ignore -b "<branch-name>" -- "<chosen-base>"`.
   - **On `<default>` with no work:** Report nothing to ship and stop.
4. **Existing PR check (ONLY when `<branch>` is non-empty):**
   ```bash
   gh pr list -R "<host>/<base_owner>/<base_repo>" --head "<branch>" --state open --json number,url,title,body,state,isDraft,headRefName,headRepositoryOwner
   ```
   - Pass the bare branch name to `--head` (never `<owner>:<branch>`).
   - Exit-0 `[]` = no open PR. Non-zero exit = **unknown** (verify `gh auth status`/connectivity before creating).
   - Match BOTH `headRepositoryOwner` AND `headRefName` (stop and show candidates on ambiguity).

---

## Step 2: Commit, Verify Publishing Gate & Push

1. **Commit uncommitted in-scope changes:** If in-scope uncommitted files exist, invoke **`ce-commit`** passing `exclude:<paths>` through (**control returns here** with the commit hash(es); if `ce-commit` is unavailable, stage named files only with `git add --` and commit via temp message file).
2. **Project publishing gate (evaluated against the exact committed state, immediately before pushing):** Check active project instructions for required pre-push checks (tests, typecheck, lint, validation). Reuse valid results whose tested inputs/environment have not changed; stop before pushing if any required check is missing or failing.
3. **Pre-existing unpushed commits guard:** Check `git log --oneline "<base-remote>/<default>..HEAD" --` (or `"@{u}..HEAD"` when an upstream exists; non-zero exit = **unknown** -> stop). If the branch carries pre-existing unpushed commits NOT created in this run and not already in an open PR: interactive -> list them and confirm before pushing; `mode:pipeline` -> stop and report a blocker.
4. **Push:** Re-verify branch and `<push-remote>`, then run `git push -u "<push-remote>" -- HEAD`.

---

## Step 3: Compose Outcome-First Title & PR Description

Write prose cleanly (following `ce-noslop` `mode:author` rules when available; control returns here). Check project PR templates (repo root, `docs/`, `.github/`, `.github/PULL_REQUEST_TEMPLATE/`) and repo instructions — project PR-body contracts win on structure.

1. **Resolve full diff range (reusing Step 1's resolved target/remotes, or resolving them here when Step 1 was skipped):**
   - **PR mode (PR ref supplied or open PR found in Step 1):** Run `gh pr view "<ref>" -R "<host>/<base_owner>/<base_repo>" --json baseRefName,headRefOid,url,body,state,isCrossRepository,headRepositoryOwner`. If `state` is not `OPEN`, report and stop. Use `<base> = baseRefName` and `<head> = headRefOid`; fetch both from `<base-remote>` (`git fetch --no-tags -- "<base-remote>" "<base>"`), falling back to `gh pr diff "<ref>" -R "<host>/<base_owner>/<base_repo>"` and `gh pr view "<ref>" -R "<host>/<base_owner>/<base_repo>" --json commits` when local git cannot reach them.
   - **Current-branch mode:** Use resolved `<base-remote>` and `<default>` as `<base>` and `HEAD` as `<head>`, fetch `"<base-remote>" "<base>"`, and inspect `git log --oneline "<base-remote>/<base>..<head>" --` and `git diff "<base-remote>/<base>...<head>" --`. Stop if zero commits exist.
2. **Build an internal Scope Map before drafting:**
   - Group commits/diff into material outcome clusters and derive ONE **umbrella outcome** stating *what is now different for users/operators* (never what files were edited or how work was done).
   - Check **program altitude** from context already in hand (plan/series/stack): if part of a larger series, note program outcome, this PR's slice, and known prior/follow-up neighbors.
3. **Size by reviewer decision cost (not line count):**
   - *Small + simple* (typo, dep bump, minor config): 1-2 sentences, no headers (<300 chars).
   - *Small + non-trivial* (bug fix, behavior change): 3-5 sentences; visible before/after for user-facing bugs.
   - *Medium feature/refactor:* 1-2 sentence opening carrying one idea + only sections answering remaining reviewer questions (design decisions, risks).
   - *Large / architectural:* Same + 3-5 design-decision callouts + concise test/validation summary (target <=100 lines, cap ~150; summary table rather than H3-per-mechanism).
   - *Performance:* Before/after markdown table naming bounds (sample size, workload, bottleneck).
4. **Title:** Match repo title conventions (project instructions > recent log > `type(scope): imperative lowercase umbrella outcome`, <72 chars, no trailing period, `fix:` over `feat:` when ambiguous; add `!`/`BREAKING CHANGE:` only with explicit confirmation).
5. **Related tracker references (preserve existing refs on rewrite):**
   - Closing syntax (`Fixes #123`, `Fixes ENG-123`) ONLY when targeting the default branch and fully resolving the issue.
   - Non-closing syntax (`Related: #123`, `Related to ENG-123`) on its own line/block — keep it separate from words like "fixes/addresses/closes".
6. **Evidence & visual aids:** Never invent evidence. Incorporate user/caller-supplied demos/screenshots (`## Demo` / `## Screenshots`); if a material visual UI change has no screenshots, state why in 1 line. Add concise validation notes for non-visual changes (never label test output "Demo"). Use Mermaid diagrams/tables when clearer than prose, full commit URLs for repo file links (relative paths 404 on PRs), and avoid starting bullets with `#`.
7. **Special sections & rewrite preservation:**
   - **On rewrite:** Preserve existing `## Demo`, `## Screenshots`, `## New concepts`, contributor credit, and branding blocks verbatim unless the user explicitly asks to refresh/remove them.
   - *Session-settled provenance:* If a plan with `(session-settled: ...)` entries is in hand, include 1 concise provenance sentence.
   - *`## Unapplied review findings`:* Splice caller-supplied checklist verbatim when provided.
   - *`## New concepts` (optional, max 2):* ONLY if the PR introduces a genuinely new, transferable pattern/library absent from `"<base-remote>/<base>"` (`git grep` against base, not working tree) — 10-25 lines (what it is, why vs. alternative, example, when not to use it).
   - *Contributor credit:* Credit other human authors (exclude bot/model/harness identities) when shipping their commits/patches. Probe `gh repo view "<host>/<base_owner>/<base_repo>" --json squashMergeAllowed -q .squashMergeAllowed`: if `true` OR if the probe fails (**unknown** — safe direction preserves credit), place exact `Co-authored-by: Name <email>` trailers as the body's **final paragraph** after any template sections.
8. **Pre-apply audit:** Verify title and opening both state the umbrella outcome, cut sentences the diff already shows, verify numbers state their bounds, and cut fluff while keeping required template sections.

---

## Step 4: Create or Update PR & Report

1. Re-verify existing PR state right before creating. Create one unique private temp directory (`mktemp -d "${TMPDIR:-/tmp}/ce-commit-push-pr-XXXXXX"`), write the body to a file inside it, and pass `--body-file "<body-file>"` (never stdin). Pass `--title` so zero characters are shell-interpreted (e.g., single-quoted with any embedded `'` escaped as `'\''`):
   - **No open PR:** `gh pr create -R "<host>/<base_owner>/<base_repo>" --title '<escaped-title>' --body-file "<body-file>"` (add `--draft` if requested; `--base "<base>"` if non-default).
   - **Open PR exists:** Confirm before rewriting its body in interactive mode (skip rewrite in `mode:pipeline` unless in description-update mode); skip `gh pr edit` if identical; otherwise run `gh pr edit "<num>" -R "<host>/<base_owner>/<base_repo>" --title '<escaped-title>' --body-file "<body-file>"`.
2. Report the PR URL, branch, commits pushed, and any excluded uncommitted files. If `## New concepts` was added, note that `ce-explain <concept>` can unpack it deeper. Once review comments arrive, `ce-resolve-pr-feedback` handles judging, fixing, replying, and resolving them.
