---
name: ce-worktree
description: "Set up isolated git worktrees — create a new branch for fresh work, or attach a worktree to an existing branch, PR, or commit. Use when starting isolated work or isolating an existing ref."
argument-hint: "[optional: PR number, branch name, or commit ref to isolate]"
---

# Worktree Isolation

Ensure the current work happens in an isolated workspace without disturbing the user's primary checkout. Most coding harnesses create a worktree at session start, so check for existing isolation first.

**Done when:** the caller is working in an isolated tree (existing or newly created) and its absolute path and branch have been reported (control returns to the caller), or a blocker has been reported instead.

**Order of operations:** verify git repo -> detect existing isolation -> prefer the harness's native worktree tool (when it supports the requested mode) -> fall back to plain git.

---

## Modes

- **New work (default):** No ref named -> create a fresh branch from the default branch on its tracking remote. Used by `ce-work` when starting isolated work.
- **Isolate an existing ref:** Caller names a PR number, branch, tag, or commit -> attach a worktree to that ref instead of creating a new branch (used by `ce-dogfood`, `ce-polish`, or PR checkout/fixes).
  - **One-branch-one-worktree rule:** Git allows a branch to be checked out in only one worktree at a time. If the named ref is already checked out anywhere (most commonly as the primary checkout's current branch), report that it is already checked out at `<path>` and let the caller act (work there in place, or create a *detached* worktree at the same commit only if a clean separate tree is essential).

---

## Step 0: Verify Git Repo & Detect Existing Isolation

First run `git rev-parse --show-toplevel` (if non-zero exit, report not-a-git-repo blocker and stop).

Compare the **physical symlink-resolved** git dir against the **physical symlink-resolved** common git dir, checking every probe's exit status explicitly (if either probe fails -> **unknown**, never claim isolation):

```bash
GIT_DIR_RAW=$(git rev-parse --git-dir) || exit 1
COMMON_DIR_RAW=$(git rev-parse --git-common-dir) || exit 1
GIT_DIR_PHYS=$(cd "$GIT_DIR_RAW" && pwd -P) || exit 1
COMMON_DIR_PHYS=$(cd "$COMMON_DIR_RAW" && pwd -P) || exit 1
```

- **Equal** -> normal checkout; continue to Step 1.
- **Different** -> either a linked worktree OR a submodule. Run `SUPER=$(git rev-parse --show-superproject-working-tree)` and check exit status (non-zero -> **unknown**, continue to Step 1 rather than claiming isolation):
  - **Non-empty** -> submodule; treat as a normal checkout and continue to Step 1.
  - **Empty (with exit 0)** -> **already isolated**. Report the worktree root (`git rev-parse --show-toplevel`) and current branch (`git branch --show-current`), then work in place (never nest a worktree inside a worktree).
    - In **existing-ref mode**: if the target ref is not already checked out here, run `git status --porcelain` and check exit status. If exit is non-zero (**unknown**) OR stdout is non-empty (**dirty**), do **not** switch branches — report the state and return to the caller. Only when exit 0 and empty, check out the target ref here.

---

## Step 1: Prefer the Harness's Native Worktree Tool

If the active harness exposes a native worktree primitive (e.g., `EnterWorktree` / `WorktreeCreate`, `/worktree`, or `--worktree`) AND it supports the requested mode (note: many native tools only create a fresh branch and cannot attach an existing ref/PR), use it, report the resulting path and branch, and return control to the caller. Otherwise proceed to Step 2.

---

## Step 2: Plain Git Fallback

Run ONLY when Step 0 found no existing isolation AND Step 1 had no suitable native tool.

1. **Move to the repo root first:** `cd "$(git rev-parse --show-toplevel)"` so `.worktrees/<slug>` lands at repo root rather than a subdirectory.
2. **Resolve base, remote & sanitized directory slug:**
   - Sanitize every worktree directory `<slug>` strictly to `[a-z0-9-]` (rejecting `/`, `\`, and `..`; fail on directory collision rather than overwriting).
   - For **new work**, resolve the default branch on its tracking remote (`<remote>/<default>`). If the default branch/remote is unknown -> interactive: ask or state you are branching from `HEAD`; non-interactive: branch from `HEAD` and disclose it. Choose a meaningful branch name (e.g., `feat/login`, `fix/email-validation`) paired with a separator-free directory slug (`feat-login`, `fix-email-validation`).
3. **Ensure `.worktrees/` is ignored without dirtying tracked files (submodule-safe):** Run `git check-ignore -q .worktrees/` (**with the trailing slash**). If exit 1 (not ignored), resolve the exclude file via `EXCLUDE=$(git rev-parse --git-path info/exclude)` (works for both normal repos and submodules where `.git` is a file) and append `.worktrees/` there.
4. **Refresh the resolved remote base (non-fatal):** When `<remote>` and `<default>` are resolved, run `git fetch -- "<remote>" "<default>"`. If offline or local-only, continue with the local ref.
5. **Create the worktree per mode (double-quote every placeholder and terminate options before positional refs):**
   - **New work:** `git worktree add -b "<branch-name>" ".worktrees/<slug>" -- "<base-ref>"`.
   - **Existing branch, tag, or commit:** `git worktree add ".worktrees/<slug>" -- "<target-ref>"` (or `--detach` for a raw commit SHA).
   - **Pull request (`#<n>`):** Resolve the PR's target `(host, owner, repo)` once and pass `-R "<host>/<owner>/<repo>"` explicitly to `gh`. Always check out on a local branch (never detached `FETCH_HEAD`):
     - When `gh` is authenticated (required for fork PRs or when pushing commits back): `git worktree add --detach ".worktrees/pr-<n>"` -> `cd ".worktrees/pr-<n>"` -> `gh pr checkout "<n>" -R "<host>/<owner>/<repo>"`. *(If it is a fork PR and `gh` is not authenticated, stop and report the blocker.)*
     - Same-repo read/local use without `gh`: `git fetch -- "<resolved-remote>" "pull/<n>/head:pr-<n>"` then `git worktree add ".worktrees/pr-<n>" -- "pr-<n>"`.
   - If git reports the ref is already checked out elsewhere, apply the one-branch-one-worktree rule above.
6. `cd` into the new worktree, report its absolute path and branch name, and return control to the caller.

**Fail closed on permission/sandbox errors:** If `git worktree add` fails due to sandbox or filesystem permissions, isolation does NOT exist. Do **not** retry alternative paths or silently work in the primary checkout:
- **Non-interactive caller** (e.g., `ce-dogfood`): return the blocker immediately to the caller without asking a blocking question.
- **Interactive:** report the failure and ask (<=2 options): work in the current checkout vs. stop and fix permissions.
