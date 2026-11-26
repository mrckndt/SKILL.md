---
name: ce-brainstorm
description: "Explore vague or ambiguous ideas into a right-sized Product Contract (WHAT to build, not HOW). Use when scoping a feature or resolving product decisions before planning. Not for executing already-specified work."
argument-hint: "[feature idea or problem to explore]"
---

# Brainstorm Workflow (Product Contract & Scope Framing)

Explore a vague or ambiguous idea through structured dialogue to produce a right-sized **Product Contract** (WHAT to build, not HOW to build it).

**Hard boundary:** Define product behavior, scope boundaries, and success criteria only. Never write implementation code, schemas, file layouts, endpoints, or technical architecture (unless the brainstorm itself is explicitly about an architectural decision).

---

## Core Interaction Rules

1. **One question per turn.** Pick the single highest-leverage question and ask only that; wait for the answer before asking the next.
2. **Default to concrete choices.** Prefer single-select multiple choice with **2-3 distinct, defensible options** (never more than 3 explicit options and no explicit `"Other"` in a blocking question tool, since hosts like Codex cap options at 3 and add their own free-text choice). When >3 options are defensible, present them in numbered chat prose and ask open-ended or split sequentially. Use multi-select only for compatible sets (<=3 options, else chat prose). Use open-ended questions for rigor probes, scoping-synthesis confirmation, and genuinely narrative/diagnostic answers, anchored on something concrete (e.g., an observed workaround or specific actor).
3. **Look up repo facts yourself.** Check the codebase, existing docs, and conventions for anything the environment can answer; reserve questions for unresolved product direction, priorities, trade-offs, and scope.
4. **Verify before claiming.** Confirm claims about existing code, tables, configs, dependencies, or absence ("no retry exists") against the repo; label anything unverified as an explicit assumption.
5. **Apply YAGNI to carrying cost.** Recommend the smallest scope that achieves the user's underlying goal without narrowing what they asked for. Recommend added scope or safeguards only when the goal fails without them or leaving them out lets uncaught harm land. Present expensive-to-add-later items (stored data, shared interfaces, money, security) as explicit choices without pushing them as defaults.
6. **Keep one coherent work unit per artifact.** When a request bundles independently deliverable outcomes, propose a plain-language breakdown with only currently supported relationships and ask which single area this brainstorm owns (record surrounding areas as tentative context). Treat multiple named providers, devices, or data sources as **coverage requirements**, splitting only when a shared access path cannot satisfy them.
7. **Settled-decision discipline:**
   - **Classification & directives:** Tag a decision as settled ONLY after a trade-off or alternative was surfaced and the user chose between options (`user-directed`) or assented (`user-approved`). Never tag an unexamined agent proposal or upgrade bare assent to `user-directed`. A cold user assertion with no examined trade-off ("build it with X") or a carried-in note lacking a rejected alternative is an unexamined **directive** (not settled): give it one challenge during dialogue. Do not re-ask settled decisions, and count them as already-probed.
   - **Contradicting evidence:** Settled decisions never suppress defect evidence. If evidence shows a settled decision is suboptimal-but-workable, keep it with a conflict call-out; if infeasible or destructive, stop and surface the blocker.
   - **Subagent briefing:** Pass discovery subagents settled decisions plus rejected alternatives (without advocacy) and `"If you find evidence a settled decision cannot work, report it; do not suppress it"`. Keep verification subagents blind to settlement markers.

---

## Phase 1: Intake, Scope Triage & Grounding

0. **Route, Resume & Fast-Path Check:**
   - **Blank invocation:** If no idea or problem was supplied (and none is active in the conversation), ask what the user wants to explore and wait.
   - **Direct answer:** Answer trivial factual questions or single-step requests directly without entering brainstorming.
   - **Verdict redirect:** If the request is really asking for a yes/no adoption verdict on a named technology or a holistic take on an existing doc, route to `ce-pov` instead.
   - **Non-software route:** If the task does not involve building, modifying, or architecting software, bypass repo scans, Grounding Scout, Claim Verifier, and Phase 4 unified-plan writing. Keep Core Rules 1-2 and the Blindspot Pass; explore (broad -> options -> converge), present an in-chat summary (decisions, direction, open threads, assumptions), and offer next steps (hand off to `ce-plan`, save as a plain markdown file, or finish).
   - **Resume:** Check `docs/plans/` for a recent requirements-only plan on this topic (has `## Product Contract` without `## Planning Contract` or `## Implementation Units`). If found, summarize its current state and ask whether to continue updating it in place or start fresh.
   - **Already-clear requirements fast path:** When the prompt already provides specific acceptance criteria, exact expected behavior, and constrained scope, classify scope tier (Step 1), run the Inline Constraint Check (Step 2) if repo-backed, skip Grounding Scout / Pressure Test / Phase 2 dialogue / Phase 3 Approaches, and proceed directly to Phase 3 Step 2 (Scoping Synthesis), preserving Path A/B tier rules.

1. **Classify scope tier & coherent-work boundary:**
   If the tier is unclear, ask one targeted question; when still uncertain, pick the heavier tier:
   - **Lightweight:** Small, well-bounded, low ambiguity -> brief repo scan, 3 internal sanity checks (real user problem? duplicating existing capability? clearly better framing at near-zero cost?), minimal dialogue, skip Grounding Scout / Approaches / Claim Verifier, ends in chat unless stable IDs are earned or requested.
   - **Standard:** Normal feature or bounded refactor with real product decisions -> full flow below.
   - **Deep:** Cross-cutting, strategic, or highly ambiguous work -> full flow below. Classify as **Deep - product** when product shape, actors, or positioning are unresolved rather than inherited.
   - **Coherent-work check (before grounding or dialogue):** Apply Core Rule 6; if the request bundles independently deliverable outcomes, agree on the single owned area before dispatching the Grounding Scout or starting Phase 2.

2. **Inline Constraint Check & Grounding Scout (Standard / Deep):**
   - **Inline first:** Read repo-root `STRATEGY.md` (or `PRODUCT.md`/`VISION.md`) for product boundaries and `CONCEPTS.md` for canonical domain terms if they exist.
   - **Grounding Scout:** Dispatch one plain generic subagent (in background if supported, so Phase 2 starts during think-time). Queue and retry on capacity backpressure; fall back inline ONLY when no subagent tool exists or a non-capacity launch failure occurs (and disclose it). Pass the topic and settled decisions per Core Rule 7. Budget ~20 targeted reads to locate similar features, prior plans/specs, adjacent examples, and current state of touched surfaces. Instruct it to return only a 3-5 line gist plus concise verbatim pointers (`file` + symbol/fact), no proposals. Wait for it before Phase 3 if still running.

3. **Internal Product Pressure Test (Standard / Deep; agent-internal, never a checklist dump):**
   Scan the user's opening (focusing scrutiny on unexamined directives; skip already-settled decisions) and fold ONLY the real gaps into Phase 2 as **separate open-ended probes (no option menus; one gap per probe)** before Phase 3. If a probe surfaces genuine uncertainty, record it as an explicit assumption:
   - **Evidence gap:** Claims a need without observable user behavior -> ask for the most concrete action or workaround someone has already taken.
   - **Specificity gap:** Vague beneficiary -> ask for a specific person/segment and what changes for them when this ships.
   - **Counterfactual gap:** Unclear status quo -> ask what users do today and what it costs if nothing ships.
   - **Attachment gap (probe last before Phase 3):** Fixated on one solution shape -> ask what the smallest version delivering the same real value looks like. Judge presence from the opening; do not skip it just because narrowing already produced a shape.
   - **Durability gap (`Deep - product`, when present):** Value rests on a shifting world -> ask how it holds under plausible near-term shifts (push past generic rising-tide answers).
   - **Product-thesis probes (`Deep - product`, ask as separate open-ended turns):** (a) what adjacent product we could accidentally build instead and why it is the wrong one; (b) what would have to be true in the world for this to fail.
   - *Internal synthesis check:* Weigh whether a nearby framing, simplification, or doing nothing is higher leverage (and on Deep, whether this is a local patch vs. system direction).

---

## Phase 2: Collaborative Dialogue

1. **Ask what the user is already thinking** before proposing your own framing. Move broad (problem, actors, value) -> narrow (constraints, exclusions, edge cases).
2. **Conflict check:** If a user's wording or claim contradicts `CONCEPTS.md` or verified repo behavior, raise the conflict before treating it as settled.
3. **Blindspot Pass (territory-scoped; trigger only on unfamiliar territory):**
   - **Trigger:** The user explicitly flags unfamiliarity with a domain/subsystem, OR 2 consecutive answers show they *cannot evaluate* the options (not merely undecided). Continue normal questions about the user's own problem, users, and evidence.
   - **Offer (blocking 2-option choice, once per flagged territory):** *"Map out the 3-7 key decisions, realistic options, hazards, and recommended defaults first, or keep going with questions where I fill gaps with labeled assumptions?"* (`Map the territory first` / `Proceed with questions`). On decline, proceed with questions and fill gaps with recommended defaults as explicit assumptions.
   - **If accepted:**
     1. **Ground first:** For in-repo territory, wait for the Grounding Scout or read the repo directly (never map in-repo code from memory). For external domains, use web search when available, or label model-knowledge items `Unverified - from model knowledge`. **Display** any questions the codebase/sources already answer as settled ground with citations.
     2. **Present the map (3-7 items, max 4 lines each):** Put architecture-changing **decisions** first (2-3 genuinely defensible options, trade-offs, plain recommended default; list ruled-out options only in why-it-matters). Put **hazards** last (hazards state constraints only, with no option menu or default).
     3. **Re-enter dialogue:** When >3 decisions are on the map, present the numbered list in chat and ask open-ended which decisions to walk through (otherwise <=3 options). If the user wants a deeper explanation of an unfamiliar subsystem first, invoke `ce-explain` (caller mode; returns here to resume dialogue). Walk through selected decisions **one per turn** as <=3-option single-select menus (or numbered chat prose when >3); record unselected decisions and hazards as explicit assumptions/constraints.
4. **Integration check (before exiting dialogue):** Mentally combine all user answers plus defaults. For each non-obvious downstream interaction the user has not tracked, ask one open-ended probe.
5. **Exit condition:** Exit when all hold **or the user explicitly wants to proceed**: primary actor known (or marked unknown), desired outcome stated, key in-scope/out-of-scope boundaries clear, success signals known or recorded as assumptions, all found rigor gaps probed or recorded as assumptions (settled decisions count as already-probed), and no integration-check question pending.

---

## Phase 3: Explore Approaches & Scoping Synthesis

1. **Propose 2-3 mechanism/product-shape approaches (Standard / Deep only):**
   - Stay at product/mechanism level (e.g., "rule property vs. separate entity"), leaving architecture (tables, file paths, classes) to `ce-plan`. At `Deep - product`, vary *what* is built (actors, positioning, product shape).
   - Include at least one non-obvious angle (inversion, constraint removal, cross-domain analogy) and optionally one higher-upside challenger option (presented alongside the baseline, never as default; omit if already over-scoped).
   - Apply an **anti-genericness test**: sharpen each option against repo grounding or drop generic listicle choices.
   - Present all approaches first (2-3 sentences, pros/cons, risks, best fit) BEFORE stating your recommendation. Recommend the smallest scope that meets the goal, noting whether it reuses, extends, or builds net-new.
   - **Selection gate:** When multiple approaches are presented, ask the user to choose one (single-select blocking question per Core Rule 2) and wait for their choice before moving to Step 2. Proceed straight to Step 2 only when one direction is clearly best and no menu is shown.

2. **Scoping Synthesis (checkpoint before writing; all tiers):**
   - Resolve any non-defaultable open question before presenting the synthesis.
   - Internally sort scope into three buckets (`Stated`, `Inferred`, `Out of scope`) to guide Phase 4 placement: `Stated` -> `Requirements` (with `session-settled` choices -> `Key Decisions`), confirmed `Inferred` bets -> `Key Decisions`, `Out of scope` -> `Scope Boundaries`, and quality/metric signals from any bucket -> `Success Criteria`.
   - Present in chat as concise collaborator prose in your own words (synthesizing understanding rather than replaying Q&A or leaking code paths; omit empty sections): **The problem** (1-2 sentences), **What we're building** (1-3 sentences), **Carrying forward:** (prior-session settled decisions only, stated as facts), and flat 1-sentence **Call outs** (bets, unconfirmed deferrals, combination effects; on Deep scope, re-check if you have zero call-outs).
   - **Path A (zero blocking questions were asked AND tier is Lightweight):** Emit only the 1-3 line "What we're building" summary (noting the user can interrupt if the shape is wrong) and proceed in the same turn to Phase 4 if a file is warranted, otherwise directly to Phase 5 Handoff.
   - **Path B (at least one blocking question was asked, OR tier is Standard/Deep even with zero questions):** Present the full scoping synthesis and ask for confirmation **open-ended (no option menu)**, naming the actual next step (writing the plan file vs. ending in chat).
   - **Revision loop:** A revision is not a confirmation. After any user change, integrate it, re-present the revised synthesis, and wait for explicit confirmation. Offer a blocking `Proceed and write` / `Hold off and keep discussing` cut only if the *same underlying decision dimension* is revised twice (new-item revisions continue freely). If the user signals they want a different workflow (e.g., jump straight to debugging or implementation), stop and redirect cleanly.

3. **Claim Verifier (parallel during Path B confirmation wait, Standard / Deep):**
   If the upcoming doc asserts checkable repo facts (especially absence claims), dispatch one fresh plain generic subagent (~15 targeted reads; keep blind to settlement markers per Core Rule 7; do not block the confirmation prompt on it) to mark each claim `confirmed`, `refuted` (with evidence), or `unverifiable`. Fix refuted claims and label unverifiable ones as explicit assumptions before writing. Queue and retry on capacity backpressure; fall back to inline verification only when no subagent tool exists or a non-capacity failure occurs (disclosing it).

---

## Phase 4: Write Requirements-Only Unified Plan (if warranted)

Write a file ONLY if decisions need stable IDs for downstream planning/review or the user asked for one (`docs/plans/YYYY-MM-DD-<type>-<topic>-plan.md`, where `<topic>` is a sanitized kebab-case slug `[a-z0-9-]` with `/`, `\`, and `..` rejected; append `-2`, `-3`, ... on filename collision so existing plans are never overwritten). Emit ONLY `## Goal Capsule` and `## Product Contract` (plus any material custom product section); leave technical planning sections to `ce-plan`. Omit empty sections, `status` fields, and `## Next Steps`.

**Structure:**
- **Metadata (YAML frontmatter):** `title` (`<Name> - Plan`, without `feat:`/`fix:` prefix), `type` (`feat` | `fix` | `refactor` | `chore` | `docs`), `date` (`YYYY-MM-DD`), `topic` (kebab-case).
- **`## Goal Capsule`:**
  - **Objective:** Outcome true for users or operators afterwards, verifiable outside the changed component (still true under a different implementation; keep constraints on owning `R` IDs). When a broader request was split, name the active area in Objective. If the prompt supplied a solution approach ("move X to Y"), place that in **Means** and derive Objective from the outcome it serves.
  - **Means** *(only if fixed)*: Chosen approach in 1 line.
  - **Product Authority:** Who/what governs product scope (when split, state that surrounding areas are not active scope).
  - **Open Blockers:** Unresolved `Resolve Before Planning` items, or none.
- **`## Product Contract`** *(render subsections below as `###` headings)*:
  - **`### Summary`:** 1-3 lines, forward-looking proposal only.
  - **`### Problem Frame`** *(when motivation is not obvious)*: Backward-looking situation and pain only; leave the remedy in Summary.
  - **`### Key Decisions`** *(provenance index when opinionated choices were made)*: Bold decision, max 1-line rationale, exact `Governs R...` links when constraining requirements (omit links for pure framing decisions). Tag user-settled entries inline: `(session-settled: user-directed|user-approved - chosen over <alt>: <reason>)`.
  - **`### Requirements`:** Continuous IDs (`R1.`, `R2.`, ...) grouped under bold inline headers when spanning distinct concerns. One sentence of intent plus at most one qualifier per R (move two-outcome forks to `Open Questions`). Commit only what the user asked for/chose, what those need to work, or what passes Core Rule 5; place unrequested safeguards in `Scope Boundaries` (deferred) or `Open Questions`.
  - **Include only when material:**
    - `### How This Work Fits Together` *(required when a broader request was split)*: Tentative surrounding areas with explicit relationship phrases (`Depends on`, `Enables`, `Can proceed independently of`), framed as current understanding rather than a committed roadmap.
    - `### Actors` (`A1.`): Multi-party behavior (humans, agents, systems).
    - `### Key Flows` (`F1.`): Multi-step behavior (note why if omitted on behavioral work).
    - `### Acceptance Examples` (`AE1.`): Required for conditional ("When X, Y") requirements.
    - `### Success Criteria`: Metrics, quality bars, or handoff signals not already stated in Requirements (avoid duplicating them).
    - `### Scope Boundaries`: Explicit non-goals and deferrals (for `Deep - product`, split into `Deferred for later` vs. `Outside this product's identity`).
    - `### Dependencies & Assumptions`: Material prerequisites and explicit assumptions.
    - `### Open Questions`: Split into `Resolve Before Planning` vs. `Deferred to Planning`.
    - `### Sources / Research`: Only breadcrumbs useful to a cold planner (code pointers, docs, prior plans).
  - **Structural Visualizations:** Add Mermaid (`TB`) diagrams next to concepts with real structural shape (lifecycle, multi-step flow, data transformation, fan-out, decision boundary, UI layout). Diagrams complement complete standalone IDed prose and never replace it.

**Format & Prose Discipline:**
- Limit IDs strictly to `R`, `A`, `F`, `AE` with plain unbolded prefixes (`- R1.`). Use bold leader labels (`**Trigger:**`, `**Covers R4.**`) inside Flows/AEs instead of H4+ headings. Use repo-relative paths, pure Markdown, and omit workflow metadata (such as Phase-X notes, timestamps, or reading prompts).
- Lead with the decision or outcome; keep at most 1 parenthetical per sentence; state each normative rule in full only on its owning `R` ID (other sections cite that ID via `Governs R...`, `Covers R...`, or `per R...` and add only section-local detail, with no unlinked duplicate rules); resolve superseded text in place rather than stacking resolution layers; bind external authorities by citing path plus deltas only.

**Ready for Planning Check (internal; run before Phase 5 whenever a file was written):**
1. **Complete:** No `TBD`/placeholders; every open question classified; `### How This Work Fits Together` present if request was split.
2. **Consistent:** No contradictions across sections; passes cold-reader Objective test (a colleague not in this session knows what this work is from Objective alone); no >1 parenthetical sentences, forked requirements, or unlinked duplicate rules.
3. **Focused:** Single coherent work unit; surrounding work stays context/non-goal.
4. **Usable by planning:** Downstream `ce-plan` can design HOW without inventing product behavior.

Fix a failed check in place ONLY when the correction preserves settled intent, then rerun the failed checks. If fixing a failure would choose or change product behavior or scope, ask one targeted question, update the artifact after the answer, and rerun the failed checks (keep blocked if the user is unavailable).

---

## Phase 5: Vocabulary Capture & Handoff (all software runs)

1. **Vocabulary Capture (optional):** If `CONCEPTS.md` already exists at repo root (creation is owned solely by `ce-compound` Full mode), silently add or refine newly resolved project-specific domain terms (entities, processes, statuses only; exclude code symbols and provisional terms; leave existing entries intact).
2. **Handoff:** Report the artifact's absolute path when a file was written.
   - **If `Resolve Before Planning` is empty:** Offer a blocking choice of at most 3 options: (1) proceed to `ce-plan` *(mention in prose that `ce-doc-review` can pressure-test the written Product Contract or `ce-prototype` can test a costly feel/UX question first)*, (2) refine scope further, or (3) finish.
     - On `ce-plan`: pass the artifact's absolute path (or concise settled-decisions summary when chat-only); **control transfers** and `ce-brainstorm` ends.
     - On `ce-doc-review` or `ce-prototype`: pass the artifact's absolute path; **control returns** here at Handoff afterward.
     - On `refine scope further`: return to Phase 2 dialogue, re-synthesize, update the artifact (if written) + rerun Ready for Planning Check, and return to Handoff.
   - **If `Resolve Before Planning` is non-empty:** Hold `ce-plan` until blockers clear. Ask the blocking questions one at a time by default. If the user prefers to pause, state what is holding planning and note they can tell you to proceed anyway (converting each remaining item into an explicit decision, assumption, or `Deferred to Planning`). After answering or overriding blockers, update the artifact in place, clear `Open Blockers`, rerun Ready for Planning Check, and present Handoff with `ce-plan` available.
