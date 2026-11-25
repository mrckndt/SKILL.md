# Intent, Drift, and Spotlight Paper Summary (Human Reference Only)

This file summarizes the research behind the `intent-review` skill for human readers. Agents do not need to read it while running the skill.

## Source

- **Title:** From Code Review to Code Critique: Intent, Drift, and Spotlight for AI-Generated Diffs at Scale
- **Authors:** Chandra Maddila, Mashrur Rashik, Euna Mehnaz Khan, Smriti Jha, James Saindon, Nachiappan Nagappan, Peter C. Rigby (Meta; Concordia University)
- **Version:** arXiv:2607.29516v2 [cs.SE], revised 14 Aug 2026 (v1: 31 Jul 2026). Preprint, no venue.
- **Abstract:** https://arxiv.org/abs/2607.29516v2
- **PDF:** https://arxiv.org/pdf/2607.29516v2
- **DOI:** 10.48550/arXiv.2607.29516
- **License:** arXiv non-exclusive distribution license. Only arXiv may redistribute the paper, so this repository links to it instead of including a copy.

```bibtex
@misc{maddila2026codecritique,
  title         = {From Code Review to Code Critique: Intent, Drift, and Spotlight for {AI}-Generated Diffs at Scale},
  author        = {Maddila, Chandra and Rashik, Mashrur and Khan, Euna Mehnaz and Jha, Smriti and Saindon, James and Nagappan, Nachiappan and Rigby, Peter C.},
  year          = {2026},
  eprint        = {2607.29516},
  archivePrefix = {arXiv},
  primaryClass  = {cs.SE},
  doi           = {10.48550/arXiv.2607.29516},
  note          = {Version 2, 14 Aug 2026}
}
```

## Thesis

Agents produce code faster than people can review it line by line. Review should move from line-by-line *review* to *critique* at two levels:

- **Forest:** why the change was made (intent), how it got there, and whether it drifted.
- **Trees:** ground-level critique of correctness, security, and performance, plus pointing a human at the few regions most worth reading.

The paper's closing question: "did the agent do what you intended, and where should you look to verify?"

## Components (system name: ARCTIC)

- **Intent:** The purpose of the change, the *why* rather than the *what*. It is inferred from the developer–agent conversation (tool calls excluded), change metadata, agent plans, and linked documents. The code is **excluded** to avoid circular reasoning. The output is a numbered list of user goals. The authors say a single-prompt (zero-shot) extractor "may be sufficient for many use cases"; a multi-turn agentic extractor is slightly more precise and costs about 6x the tokens.
- **Backtranslation:** An agent turns the diff into a numbered list of tasks. It deliberately does **not** see the summary, test plan, or conversation, because those feed intent and "would bias the drift computation".
- **Drift:** Each intent item is marked fully met, partially met, or unmet. There are two kinds of drift:
  - **DNF** (Did Not Follow instructions)
  - **UC** (Unrequested Changes), which can be beneficial (an added null check) or harmful (scope creep, sensitive files)

  Only risky or clearly out-of-scope UCs are penalized. The rubric:
  - Perfect Alignment (0–10): all intents fully satisfied, no bad unrequested changes.
  - Minor Drift (11–25): all intents satisfied with only trivial deviations.
  - Moderate Drift (26–50): most intents addressed but some only partially satisfied, and/or noticeable bad UCs.
  - Significant Drift (51–75): only some intents satisfied, and/or substantial bad UCs that add risk.
  - Major Drift (76–100): no intents satisfied, near-complete deviation, and/or severe bad UCs such as deleting functionality or breaking changes.
- **Spotlight:** A non-agentic, two-stage LLM reviewer. It sees the diff, its title and summary, the modified files' contents, and framework rules, but not the agent trajectory. A generation pass proposes candidate regions. A critic pass keeps only those that pass five checks: claim-correctness, intent, framework rules, actionability, and a senior-engineer acceptance bar. The top regions are shown by severity, each with a category and a rationale. It favors correctness, security, and performance over style and best-practice nits.
- **UI:** A plain-prose "Change Structure" paragraph names unrequested additions. It is followed by a goal checklist (✓/✗), the drift bucket, spotlight cards, and key milestones from the agent trajectory.

## Key Results

- **AI vs. human reviews** (2,000 each): AI reviewers under-cover security (−89.5%), correctness (−42.6%), and performance (−59.1%). They over-produce best-practice (+327.8%) and design (+350%) comments.
- **Intent extraction:** F1 of 0.86 (agentic) and 0.84 (zero-shot) on 121 diffs. 90.2% of intent summaries were approved in production (n=112).
- **Drift scoring:** QWK of 0.907 against human raters overall. Perfect and Major buckets are reliable (F1 0.89 and 0.81). **Moderate and Significant are weak** (F1 0.34 and 0.41).
- **Effect on authors:** drift fell by 11.8 points between diff versions. Authors who saw the score cut it by a further 5.8 points (small effect, quasi-experimental).
- **Spotlight vs. baseline AI reviewer:** On problematic diffs, it is 2.4x better at deciding whether a diff needs a comment and 3.3x better at localizing defects. Across all diffs, the comment-decision gain is about 1.3x (QE 0.594 vs. 0.454). It uses 5x fewer tokens and is 6x faster. Absolute precision and recall stay modest (about 0.1 on problematic diffs).
- **Self-review path:** low-drift diffs with no spotlight findings can land after author attestation. Reverts were 0.33% vs. a 0.17% baseline, which the authors call acceptable for early access.

## Limitations Stated by the Authors

- One company, one codebase, one review culture.
- LLM-as-judge is used throughout and may have systematic biases. Bucket boundaries are subjective.
- The drift study is quasi-experimental with self-selection.
- Filtering CRBench to comments determinable from the diff may inflate precision.
- The data cannot be released, even anonymized.

## Our Observations

- Intent ground truth comes from a single annotator.
- The paper publishes no prompts and has no appendix.
- It does not address non-expert readers; all users are engineers.

## How `intent-review` Uses It

| Paper | Skill |
|---|---|
| Intent from conversation, plans, and metadata, never from code | `review.md` Step 1: Goal List built before reading the diff |
| Blind backtranslation | `review.md` Pass B: a subagent sees only the diff |
| Met / partial / unmet, DNF and UC, harmful UCs only | `review.md` Step 3 |
| Five drift buckets | `review.md` Step 3, as informational labels only; no numeric score, and the decision uses goals, harmful UCs, and findings because the middle buckets are unreliable |
| Generator + five-criterion critic | Passes A and C generate; `review.md` Step 4 applies the critic |
| Cheap, focused critic | Passes scaled to the risk tier; `high` effort only for High risk |
| Prioritize correctness, security, performance over nits | `SKILL.md` Core Operating Rule 7 |
| Change Structure paragraph and goal checklist | "In plain words" and "Did it do what you asked?" output sections |
| Streamlined path for low-drift, no-finding diffs | "No blocking issues" decision, worded as not a sign-off given the higher revert rate |

Additions not from the paper:

- the plain-language reader profile, which is grounded in Anthropic's study on AI assistance and coding skills (arXiv:2601.20245);
- test and gate integrity checks, which address reward hacking and test tampering by coding agents;
- prior-commit guarantees;
- the pre-code plan review and contract;
- orchestration of the Claude Code and Cursor built-in reviewers.
