---
name: ce-polish
description: "User-directed UX polish loop on a working feature: safely start the dev server on the feature branch, iterate on the user's live observations via hot reload, and commit locally when done. Use only when the user asks to polish a working feature live."
argument-hint: "[PR number, branch name, or blank for current branch] [--port PORT]"
disable-model-invocation: true
---

# Live UX Polish Loop

Put an already-working feature in front of the user and turn their live visual and interaction observations into tight, focused UX fixes.

**Done when:** the user signals they are done, every requested tweak is applied (or reported blocked), and polish edits are committed locally via **`ce-commit`**.

**Hard boundaries:**
- **User-directed scope:** Act only on observations the user gives (use `ce-dogfood` for autonomous QA).
- **Feature branch required:** Work only on a named feature branch (never on the default branch or detached `HEAD`).
- **Local commits only:** Commit polish edits locally at close; leave pushing and PRs to the user.

---

## Step 1: Verify Safe Feature-Branch Workspace

1. If a PR or branch name was given: invoke **`ce-worktree`** for that target ref (control returns here with the isolated path and branch; act on its verdict if already checked out elsewhere). If `ce-worktree` created a fresh worktree, check whether dependencies (`node_modules`, vendor) and `.env*` files are present; if missing, ask the user how to provision them before starting the server. If blank, stay in the current checkout.
2. Confirm `git branch --show-current` is non-empty (not detached `HEAD`) and is NOT the default branch on its tracking remote (if the default branch is **unknown**, stop and ask the user to confirm or create a feature branch).
3. Record pre-existing dirty paths (`git status --short --untracked-files=all`) as `pre_polish_dirty` (non-zero exit = unknown -> stop) so Step 4 keeps them completely out of polish commits.

---

## Step 2: Start & Verify the Dev Server

1. **Resolve start tuple (command, working dir, env, port):** Check repo config (`package.json`, `Procfile`/`bin/dev`, `.env*`, project instructions, or `--port`). If ambiguous across multiple apps/scripts or unknown, ask the user once (<=3 options or open-ended) instead of guessing.
2. **Respect port ownership:** Inspect `<port>` (`lsof -i ":<port>" -sTCP:LISTEN`; non-zero/failed probe = unknown -> ask):
   - If a process is already listening AND its listener PID's CWD resolves via `git -C "<pid-cwd>" rev-parse --show-toplevel` to the **exact same worktree top-level** as this checkout -> reuse it.
   - If occupied by an unrelated or unattributable process -> leave it untouched and ask (<=3 options): pick another port, wait for the user to stop it, or stop.
   - If free -> start the dev server in the background logging to OS temp (`mktemp -d "${TMPDIR:-/tmp}/ce-polish-XXXXXX"`).
3. **Confirm attributed reachability:** Poll `http://localhost:<port>` (or the actual URL printed by the server) for up to 30s and confirm the response comes from the selected server instance (matching its PID top-level or probe entry in its log). If unreachable, show the last 20 log lines (only for servers this run started) and ask how to adjust.
4. Open the URL in the harness's browser capability if available (best-effort convenience), then announce and wait:
   ```text
   Dev server running on <verified-actual-url>
   Browse the feature and tell me what could be better.
   ```

---

## Step 3: Iterate on Live Observations

For each observation the user gives:
1. Inspect only the minimal files governing that UI surface.
2. Apply the focused UX/copy/layout/interaction edit and let hot reload refresh the page (keep edits strictly scoped to the requested polish).
3. Only when the user asks you to visually inspect the result, check it via the active harness's browser/screenshot capability (or ask what they see if none exists).
4. Confirm briefly what changed and wait for the next observation.

---

## Step 4: Close Locally

When the user says they are done:
1. Subtract `pre_polish_dirty` from the list of polish-modified paths before committing; report any overlapping pre-existing dirty file as left uncommitted rather than sweeping user edits into the commit.
2. Invoke **`ce-commit`** passing `files:<comma-separated clean polish paths>` and `exclude:<comma-separated pre_polish_dirty paths>` (control returns here) to create clean local commit(s) for polish edits only.
3. Report the commit hash(es), any overlapping dirty files left uncommitted, the still-running dev server URL (so they can keep inspecting or stop it), and any unresolved blocker.
