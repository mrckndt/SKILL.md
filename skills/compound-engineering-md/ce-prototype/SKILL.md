---
name: ce-prototype
description: "Build a throwaway interactive or visual prototype to settle how a feature should work, feel, or read before implementing it, then write settled decisions back to the Product Contract or hand off to ce-brainstorm/ce-plan."
argument-hint: "[question, brainstorm path, or plan path]"
---

# Throwaway Prototype

Build a throwaway prototype at the fidelity required to settle how something should **work, feel, or read** before locking in costly implementation decisions.

**Core rule — never fake the dimension under test:**
- Settled by **driving** (flow, state model, control behavior) -> build interactive state you can actually click through, not a static mockup.
- Settled by **seeing** (layout, typography, density, visual hierarchy) -> build to real visual finish (apply the Craft Floor below), not a crude wireframe.
- The **user's own perception** settles the question, never your opinion of the artifact.

**Hard boundaries:**
- Requires a human to experience the prototype; if invoked headless/`mode:pipeline`, stop and report that a human is required.
- Throwaway means unshipped and unmaintained — keep prototype code off product branches, restore overlay edits cleanly from backup, and leave kept prototype directories intact for the user to prune.
- Route WHAT-to-build scoping to `ce-brainstorm` and working-feature UX refinement to `ce-polish`.

---

## Phase 1: Scope the Question

1. **Locate the question:** Read the prompt, conversation, and any supplied brainstorm/plan. If repo context is needed, inspect ONLY the specific surface/component/constraints the question touches. Ask only for missing surface or hard constraints (must keep / must not change); never ask the user to invent the UX answer in chat.
2. **Prioritize & classify Narrow vs. Wide:** Start with the question most expensive to get wrong (combine parts when testing how they work together; skip questions already settled in the supplied doc):
   - **Narrow:** Specific detail with close options (control choice, placement, transition) -> put 2-3 close variants on one comparison surface.
   - **Wide:** Open exploration ("make this feel cleaner / explore workflows") -> **diverge first** by presenting 3-5 genuinely distinct mechanisms/organizing principles in **numbered chat prose** (not a >3-option blocking question; 1 plain line each describing what the user sees/does; palette/font swaps over the same layout do NOT count as separate avenues). Let the user reply in chat which subset to build rather than building 1 idea as the answer.
   - If narrow vs. wide is unclear, ask once (2 options).
3. **Go-ahead check:** If building wasn't explicitly authorized or the scope/split could surprise the user, state briefly what you will build and why, and wait for approval.

---

## Phase 2: Build & Preview

1. **Choose substrate & root:**
   - **Default:** Standalone web prototype (HTML/CSS/JS/SVG) even for non-web apps, unless the user named another stack or the browser cannot render the dimension honestly (if so, name the medium upfront).
   - **Storage:** Sanitize `<slug>` strictly to `[a-z0-9-]` (rejecting `/`, `\`, `..`). Prefer `.context/ce-prototype/<YYYY-MM-DD>-<slug>/` ONLY after verifying `.context/` is gitignored (`git check-ignore -q .context/` — trailing slash required; offer to add `.context/` to `.gitignore` before creating); otherwise create a unique private temp dir via `mktemp -d "${TMPDIR:-/tmp}/ce-prototype-XXXXXX"` (noting best-effort temp retention).
   - **Throwaway overlay (exception):** Touch the real app tree ONLY when explicitly asked or when testing density/chrome inside an existing screen. **Before the first overlay edit, copy every modified target file into `<run-dir>/overlay-backup/`** (preserving relative paths) AND record any **newly created file paths** in `<run-dir>/overlay-created.txt`. Never commit overlay edits; when the try ends, restore modified files strictly from `<run-dir>/overlay-backup/` (never `git checkout --`, which wipes pre-existing user WIP), delete every newly created file listed in `overlay-created.txt`, and report any file that could not be restored/removed cleanly.
2. **Craft Floor (apply ONLY on seeing questions, for dimensions actually in play):**
   - Contrast >=4.5:1 body / >=3:1 large text, controls, and focus indicators; tint secondary text from its surface.
   - Body line measure 65-75ch; one consistent spacing rhythm; clear type-scale hierarchy; visible keyboard focus.
   - Real content at shortest and longest plausible values across active states (hover, focus, error, empty, disabled) — never lorem ipsum sized to look pretty.
   - **Anti-template specificity check:** Reject default SaaS boilerplate (generic hero + 2 buttons, row of equal icon-circle cards, gradient headline, tiny uppercase labels above every section, identical border-radius everywhere). If stripping the product name makes it unrecognizable as this product, rework it.
3. **Comparison surface, visual check & local preview:** Place comparable options side-by-side or via clean variant tabs (give scroll/transition experiments full-size space). Serve strictly on loopback scoped to the run directory:
   ```bash
   python3 -m http.server <port> --bind 127.0.0.1 --directory "<run-dir>"
   ```
   Record the server PID so you stop it at Phase 4 close. **Inspect/screenshot each rendered variant at rest before sharing the URL** to verify the Craft Floor (if no browser/screenshot tool is available, say so explicitly). Share the URL in 1 line and revise in place as the user gives feedback.

---

## Phase 3: Track Decisions (`decisions.md`)

Maintain `<run-dir>/decisions.md` as the continuity capsule (recording choices once the user has genuinely judged and settled them):
- Question tested, what was built (paths/screens), winner + rationale, rejected options, stated adjustments not yet built, and remaining open questions.
- After each decision, re-evaluate whether remaining prototype questions are still worth building (stop early if the user's choice changed *what* they want to build rather than *how* it feels).

---

## Phase 4: Close, Write-Back or Hand Off

Stop the background preview server (first verifying via `ps -p <pid>` that the PID still belongs to the started `http.server` before signalling it) and restore any overlay backup + remove newly created overlay files before finalizing.

When the user applies decisions:
- **If a directly related brainstorm/plan path exists** (passed on invoke or explicitly named this session; fail closed to chat recap if none, ambiguous, or lacking `## Product Contract` — never hunt the repo or mint a new file):
  1. **Confine content edits to `## Product Contract`:** Allocate next unused `R<N>` (and `AE<N>` for state-dependent behavior), add/update `### Key Decisions` tagged `(session-settled: user-directed|user-approved - chosen over <alt>: <reason>)` with `Governs R...`, and resolve superseded text in place.
  2. Cite the kept prototype's absolute run dir and `decisions.md` under `### Sources / Research` (labeled machine-local/best-effort; requirements govern on conflict; omit for undone overlays).
  3. **Stale HOW sections:** If the file already contains `## Planning Contract`, `## Implementation Units`, `## Verification Contract`, or `## Definition of Done`, mark them visibly stale (calling for `ce-plan` regeneration) rather than deleting them silently; delete those HOW sections wholesale ONLY when the user explicitly confirms deletion (leave regeneration to `ce-plan`).
- **If no related plan exists:** Recap settled decisions, rejections, adjustments, and the kept prototype path cleanly in chat.
- **Next step:** If invoked by a calling skill (`ce-brainstorm`, `ce-plan`), return `decisions.md` and control directly to the caller. Otherwise recommend **`ce-plan`** (after write-back or when ready to plan) or **`ce-brainstorm`** (when product-level questions remain).
