---
name: ce-strategy
description: "Create or update repo-root STRATEGY.md through a structured pushback interview and stress test. Use when starting a product, adding a strategy doc, or changing direction or roadmap. Downstream skills (ce-ideate, ce-brainstorm, ce-plan, ce-dogfood) read it as product grounding."
argument-hint: "[optional: section to revisit, e.g. 'metrics', 'positioning', 'tracks', or 'boundaries']"
---

# Product Strategy (`STRATEGY.md`)

Author or update `STRATEGY.md` at the repository root — a compact, under-5-minute anchor capturing what the product is, who it serves, how it wins, and where investment goes. Downstream skills (`ce-ideate`, `ce-brainstorm`, `ce-plan`, `ce-dogfood`) read it automatically when present.

**Hard boundaries:**
- **Anchor, not plan or backlog:** Suggest `ce-brainstorm` for features, `ce-plan` for implementation plans, and the issue tracker for schedules while continuing this interview (do not transfer out mid-interview unless the user asks).
- **User answers; repo only grounds the question:** Use repo evidence to ask sharper questions or surface contradictions; capture strategy solely from the user's answers.
- **File ownership test (apply before editing any existing `STRATEGY.md`):**
  - *Solely house-format:* Every `##` heading belongs to the template set (`Purpose`, `Positioning`, `Users`, `Boundaries`, `Key metrics`, `Tracks`, `Milestones`, `Brand`) or legacy equivalents (`Target problem` -> `Purpose`, `Our approach` -> `Positioning`, `Who it's for` -> `Users`, `Not working on` -> `Boundaries`, `Marketing` -> `Brand`), with no HTML comment marker from another tool. Maintain these in house order and rename legacy headings on write.
  - *Multi-writer:* Any foreign `##` heading, prose-only shape, or other tool's marker -> read by meaning, edit minimally in its own idiom, and keep existing section order intact.
  - *Protected, unowned, or unknown ownership:* Leave `author-approved` sections, docs not owned by the user, and files whose ownership cannot be confidently classified untouched — report the conflict or write a separate linked file.
- **One question at a time:** Ask via blocking question tool (<=3 explicit options, no explicit `"Other"`) or free-form chat, preferring free-form responses for strategy sections. Cap pushback at **2 rounds per section** (after 2 rounds, capture what the user gave and flag it in chat as worth revisiting).

---

## Phase 0: Ground & Route

1. **Build a quick repo model** (bounded read): inspect existing `STRATEGY.md`, `README.md`, `CONCEPTS.md`, sibling docs (`VISION.md`, `PRODUCT.md`), top-level layout, and recent commits/PRs (recent activity informs only the Tracks question and drift checks).
2. **Present the repo model (3-5 lines)** citing sources (what the product appears to be, who it serves, where recent attention went) and invite correction. If the repo model does not establish `<Product Name>`, ask for it here. If empty/greenfield, note that in 1 line and proceed ungrounded. If a legacy sibling doc (`VISION.md`/`PRODUCT.md`) exists and `STRATEGY.md` does not, ask whether to link to it (deferring shared principles) or fold it in.
3. **Route:** No `STRATEGY.md` -> Phase 1 (First-Run Interview). `STRATEGY.md` exists -> Phase 2 (Update Run).

---

## Phase 1: First-Run Interview (with Pushback)

Apply real pushback (quote the user's own words verbatim; ask sharper questions without lecturing on anti-pattern labels; keep answers to 1-3 sentences):

1. **Purpose:** *"What's the core problem this product solves — and what makes it hard?"*
   - Push back on goals-as-problems ("grow revenue"), vague wishes, symptoms ("churn"), civilization-scale scope, or missing-feature framing. Capture 1-2 sentences of user situation + crux with zero solution language.
2. **Positioning:** *"Given that problem, what's your guiding choice or principle that makes it tractable?"*
   - Push back on generic values ("move fast"), feature lists, product descriptions any competitor could claim, multiple competing approaches, or approaches disconnected from Purpose. Capture 1-2 sentences naming the distinctive choice.
3. **Users:** *"Who is the primary user, and what job are they hiring this product to do?"*
   - Push back on "everyone" / too many primary personas, demographics instead of situations, or generic jobs ("be productive"). Capture one primary persona + concrete JTBD verb phrase.
4. **Key metrics (3-5):** *"What 3-5 metrics tell you whether this approach is working?"*
   - Push back on vanity/monotonic totals ("total signups"), >5 metrics, team outputs ("deploys/week"), or unmeasurable slogans ("delight"). Capture 3-5 leading + lagging metrics that can regress, plus where each is measured (not today's numbers).
5. **Tracks (2-4):** *"What 2-4 investment areas are you committing to?"*
   - Push back on feature checklists, >4 tracks (starved focus), single-track (no choice), vague tracks ("improve product"), or tracks disconnected from Positioning. Mention recent commit/PR focus as a question ("recent work clustered on X — track or one-off push?"). Capture 2-4 named domains + 1-line link to Positioning.
6. **Stress Test (mandatory before Boundaries, and whenever Boundaries is re-interviewed):** Pose **3-5 concrete fault-line proposals**, one per turn (1 round per proposal), where a reasonable team could go either way (tempting adjacent feature, competing persona pull, track conflict, metric gaming):
   - Strategy already decides it the same way -> confirmed.
   - Strategy cannot decide it or disagrees -> sharpen Positioning or Tracks in place (never add new sections or metrics to absorb a proposal).
   - User resists it -> candidate for **Boundaries**.
7. **Boundaries (written on first run and in house-format files; added on update only with approval):** *"What tempting things have you explicitly decided NOT to do right now?"* Offer candidates resisted in this run's Stress Test plus any boundaries already in the file; write 1 sentence per item (or `"Nothing named yet."`). Draft `_Resist a change when:_ <rule>` ONLY when at least one proposal was resisted; omit the line if none were resisted.
8. **Milestones (optional, skip by default):** External dated anchors only (launch, renewal, conference).
9. **Brand (optional, skip by default):** One-liner / 2-3 line key message.

---

## Phase 2: Update Run

1. Summarize current `STRATEGY.md` in 3-5 lines.
2. **Check drift** across every section against current repo structure, recent commits/PRs, `docs/plans/`, and `docs/solutions/`. Surface suspected stale sections with evidence as **candidates**, not verdicts.
3. Map legacy focus aliases (`approach` -> `Positioning`, `who it's for` -> `Users`). Revisit the section named in the prompt, or ask which section to revisit — if <=3 drift candidates exist, offer those (<=3 options, no explicit `"Other"`), otherwise list all sections in chat and ask open-ended. Re-interview chosen sections with full Phase 1 pushback (running the Stress Test whenever Boundaries is revisited) while leaving untouched sections verbatim. In solely house-format files, offer to add `Boundaries` if missing (add only on approval).

---

## Phase 3: Draft, Approve & Write

1. Assemble `STRATEGY.md` in the user's own language (omit empty optional sections `Milestones` / `Brand`; keep `Boundaries` per Phase 1 Step 7; set YAML frontmatter `name` and `last_updated: YYYY-MM-DD` on house-format files; verify <=4 sentences in each prose section `Purpose`/`Positioning`/`Users`, 3-5 metrics, 2-4 tracks, Purpose connected to Positioning):
   ```markdown
   ---
   name: <Product Name>
   last_updated: <YYYY-MM-DD>
   ---

   # <Product Name> Strategy

   ## Purpose
   <1-2 sentences: user situation + crux>

   ## Positioning
   <1-2 sentences: distinctive guiding choice>

   ## Users
   **Primary:** <Persona> - <one-sentence JTBD>

   ## Boundaries
   - <tempting item decided against, or "Nothing named yet.">
   _Resist a change when:_ <single decision test; omit if nothing was resisted>

   ## Key metrics
   - **<Metric>** - <1-line definition; where measured>

   ## Tracks
   ### <Track Name>
   <1-line investment area>
   _Why it serves the approach:_ <1-line link>

   ## Milestones  <!-- optional; delete if unused -->
   - **<YYYY-MM-DD>** - <external milestone>

   ## Brand       <!-- optional; delete if unused -->
   **One-liner:** <pitch>
   ```
2. Show the full draft in chat, offer an edit pass, and write `STRATEGY.md` ONLY after approval (if the user declines to save, write nothing and leave the draft in chat only).
3. Close in 1 line noting `STRATEGY.md` is read by `ce-ideate`, `ce-brainstorm`, `ce-plan`, and `ce-dogfood` (suggesting `ce-ideate` or `ce-brainstorm` if starting fresh).
