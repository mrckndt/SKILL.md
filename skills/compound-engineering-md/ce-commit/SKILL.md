---
name: ce-commit
description: "Create clean, convention-aware local git commit(s) with explicit path-limited staging and outcome-focused messages. Local commits only; use ce-commit-push-pr to push and open a PR."
argument-hint: "[files:<comma-separated paths>] [exclude:<comma-separated paths>] [commit guidance]"
---

# Local Git Commit

Create well-crafted local commit(s) from the current working tree without pushing or opening a PR (use `ce-commit-push-pr` for pushing/PRs).

**Done when:** every logical in-scope change is committed via explicit file lists with outcome-first messages, and `git status` is clean of those files (or reports nothing in scope to commit). When invoked by a calling skill, control returns to the caller with the created commit hash(es) and any excluded files.

**Scope definition:** Start from `files:<paths>` (comma-separated or repeated token) if supplied, else all modified/untracked files in `git status --short --untracked-files=all`. Always subtract any `exclude:<paths>` (comma-separated or repeated token) from the resulting set — even when `files:` was provided — and leave excluded and out-of-scope files untouched in the tree and index.

---

## Step 1: Gather Git Context (Fresh Snapshot)

Run each git command as its **own** shell call (program + args only; no `&&`, `;`, pipes, or redirects) and interpret non-zero exits directly:
1. `git status --short --untracked-files=all` -> if not a git repo (non-zero exit), stop; if nothing in scope exists, report "nothing to commit" and stop.
2. `git diff HEAD` -> inspect changes (handle unborn repo with no commits yet).
3. `git branch --show-current` -> empty means detached `HEAD`.
4. `git log --oneline -10` -> recent commit convention.
5. Resolve the default branch on its tracking remote (`<default>`). Treat a non-zero or empty probe as **unknown**.

Re-check branch and staged set right before committing if anything changed.

---

## Step 2: Branch Safety

If on detached `HEAD`, on the resolved default branch, OR when the default branch is **unknown**, automatically create a descriptive feature branch from the change content (`git checkout -b "<branch-name>"`), pick a non-conflicting suffix if taken, and verify with `git branch --show-current`. Stay off detached `HEAD` and the default branch unless explicitly directed.

---

## Step 3: Commit Structure & Message Rules

1. **Match repo conventions:** Project commit instructions in context > recent `git log` pattern > conventional commits (`type(scope): description`).
   - With conventional commits, when `fix:` and `feat:` both plausibly fit, default to **`fix:`** (remedying broken/missing behavior); reserve `feat:` for genuinely new capabilities (user override wins).
   - Add `!` or `BREAKING CHANGE:` ONLY after explicit user confirmation.
2. **Logical splits (file level only):** Make one commit per cleanly separable concern (at most 3 commits; never `git add -p`). When ambiguous, make 1 commit.
3. **Outcome-first message:**
   - Imperative subject naming **what is now fixed or possible**, not file lists (`Fix double-submit on checkout`, not `Update checkout.ts`).
   - Add a brief body ONLY when motivation or trade-offs are not obvious from the subject.
   - If a single plan Implementation Unit ID (`U<N>`) is already in hand for this commit, append `(U<N>)` to the subject (omit if spanning units or no plan is in hand).
   - If committing someone else's work (ported from their branch/PR/patch), add exact `Co-authored-by: Name <email>` trailers for BOTH the ported commits' authors AND any `Co-authored-by` trailers they carry (never guess an email; credit by name in the body when no email exists).

---

## Step 4: Path-Limited Stage, Commit & Confirm

- Always stage exact named files double-quoted after `--` (never `git add -A` or `git add .`).
- Create a unique private temp directory with `mktemp -d "${TMPDIR:-/tmp}/ce-commit-XXXXXX"` and write each commit message file inside it using your file-write tool (so `$`, backticks, quotes, and newlines pass literally with zero shell expansion), then run staging and commit as two separate calls:
  ```bash
  git add -- "<file1>" "<file2>"
  ```
  ```bash
  git commit -F "<message-file>" -- "<file1>" "<file2>"
  ```
  *(The trailing `-- "<file1>" "<file2>"` guarantees unrelated pre-staged index entries stay out of the commit.)*
- Run `git status --short` and report each commit short hash, subject, branch, and any excluded or untouched files.
