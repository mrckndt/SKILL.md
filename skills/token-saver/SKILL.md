---
name: token-saver
description: Toggle session-scoped telegraphic compression on direct chat replies via single-option commands (status, on, off). Manual invocation only.
disable-model-invocation: true
metadata:
  paper_abs: "https://arxiv.org/abs/2606.24083"
  paper_html: "https://arxiv.org/html/2606.24083v1"
---

# Token Saver

Input: `$ARGUMENTS`
Skill directory: `${CLAUDE_SKILL_DIR}`

## Parse Command

Read `Input` above if substituted; if unsubstituted, take the text after `token-saver` in the user message. Trim outer whitespace, lowercase, and accept at most one whitespace-delimited token:

- Empty or `status`: run **Status**.
- `start`, `enable`, or `on`: run **Enable**.
- `stop`, `disable`, or `off`: run **Disable**.
- Anything else (unknown token or extra words after a valid option): run **Reject**.

Resolve `<skill-dir>` from `Skill directory` if expanded; otherwise use the directory containing this `SKILL.md`.

## Mode Lookup

Resolve mode in this exact order:

1. If a `token-saver: on|off (id <id>)` line is visible in session context, take the latest `<id>` and run `cat "${TMPDIR:-/tmp}/token-saver-<id>" 2>/dev/null`; return `on` for `enabled` or `off` for `disabled`.
2. If step 1 yields neither, use the latest valid `token-saver` transition visible in session context (`on` or `off`).
3. Otherwise return `off` without creating or inspecting any file.

Pass only these fixed commands and numeric `<id>` to the shell; never pass user text. If any shell command fails, continue from session context without error.

## Actions

- **Status:** Run Mode Lookup. When `off`, leave `references/l1-telegraphic.md` unread. When `on` and its rules are missing from context, read `<skill-dir>/references/l1-telegraphic.md` and apply them to direct chat replies. Print `token-saver: on` or `token-saver: off`, adding `(id <id>)` whenever an id is known.
- **Enable:** Read `<skill-dir>/references/l1-telegraphic.md` and apply its rules to direct chat replies. Reuse the `<id>` from the latest `token-saver` status line or run `date +%s`, then run `echo enabled > "${TMPDIR:-/tmp}/token-saver-<id>"`. Print `token-saver: on (id <id>)` (omit `(id <id>)` only if no id is available).
- **Disable:** If the latest `token-saver` status line shows an `<id>`, run `echo disabled > "${TMPDIR:-/tmp}/token-saver-<id>"`; else skip writing. Stop applying `references/l1-telegraphic.md`; for all subsequent chat replies in this session, write in full, grammatical English sentences following normal session verbosity. Confirm in full, grammatical English sentences including `token-saver: off (id <id>)` (or `token-saver: off` if no id is visible).
- **Reject:** Change no state, write no file, and ignore trailing text. Run Mode Lookup without reading `references/l1-telegraphic.md` when `off`. Print current status (same format as Status) and valid single-option arguments: empty/`status`, `start`/`enable`/`on`, `stop`/`disable`/`off`.

Leave `references/cavewoman-paper.md` unread during execution; it is background for human readers only.
