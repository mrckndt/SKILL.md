# CAVEWOMAN Paper Summary (Human Reference Only)

This file summarizes the research behind the `token-saver` skill for human readers. Agents leave this file unread during runtime skill execution.

## Source

- **Title:** CAVEWOMAN: How Large Language Models Behave Under Linguistic Input and Output Compression
- **Authors:** Morayo Danielle Adeyemi, Ryan A. Rossi, Franck Dernoncourt
- **Abstract:** https://arxiv.org/abs/2606.24083
- **HTML version:** https://arxiv.org/html/2606.24083v1

## Evaluation Protocol

The paper evaluates linguistic compression across two channels on the same benchmark items:

- **Condition A (input compression):** A deterministic POS-tag filter compresses the user prompt while the system prompt stays neutral.
- **Condition B (output compression):** The user prompt stays intact while a per-level system prompt instructs the model to respond in a compressed register.

Both channels are tested across five levels (L0 Unconstrained, L1 Telegraphic, L2 Keyword-Only, L3 Noun-Phrase Skeleton, and L4 15-token budget: truncating the L3-filtered prompt to 15 tokens in Condition A and instructing a <=15-token reply in Condition B, both under a 20-token decoder ceiling; Section 3.1, Table 19, Appendix H), five datasets (GSM8K, BoolQ, ARC-Easy, CommonsenseQA, MMLU-STEM), and eight models: four API models (GPT-4o, GPT-5.4, Claude Haiku 4.5, Claude Sonnet 4.6) and four open-weight models (Qwen2.5-VL-7B, Qwen3.5-9B, DeepSeek-R1-Distill-Qwen-7B, Gemma-4-E4B). All runs use greedy decoding with per-level `max_new_tokens` budgets (L0 400, L1 300, L2 200, L3 150, L4 20; Table 7).

## Pricing Context (Introduction)

Output tokens are typically priced 4-8x higher than input tokens.

## Paper Findings

- **Finding 1, cost asymmetry between channels:** Output compression at L1 (Condition B) cuts realized cost by 1.4x-2.4x on the three API models whose billed output matches visible output (GPT-4o, Claude Haiku 4.5, Claude Sonnet 4.6), up to 3x on the best cell, cheaper on 17 of 20 API (model, dataset) cells, and cheaper on all four open-weight models under projected May 2026 public-tier pricing (1.17x-2.5x). GPT-5.4 is cheaper on only 2 of its 5 benchmarks. Input compression at L1 (Condition A) raises net cost instead (up to 1.15x on the five-benchmark mean, which is GPT-5.4's figure, and up to 1.8x on the worst dataset; Table 3 net cost: +3.1% Haiku 4.5, +5.1% Sonnet 4.6, +15.4% GPT-5.4, -2.6% GPT-4o) because models compensate with longer responses while accuracy degrades.
- **Finding 2, surface-text divergence:** Across the six-model panel whose visible L0 answers are full prose (GPT-4o, GPT-5.4, Claude Haiku 4.5, Claude Sonnet 4.6, Qwen2.5-VL-7B, Gemma-4-E4B; called the "six non-reasoning models" in Section 4.2 because GPT-5.4's reasoning tokens stay hidden server-side; Section 3.6, Appendix E), 51.9% of L1 output-compression generations are correct yet their surface text no longer entails the model's unconstrained L0 reference (80.4% under length-matched re-scoring). The paper calls this a surface-text observation, not a propositional claim.
- **Finding 3, model-specific robustness:** Relative robustness (telegraphic output vs. input accuracy) spans 1.5x to 3.7x across models and is not predicted by parameter count or unconstrained accuracy.

## Deeper Levels (Figure 3 and Table 11, Condition B)

Under output compression, classification accuracy holds through deep levels on several models, while losses on others are large. At L3, GPT-4o, Claude Sonnet 4.6, and GPT-5.4 lose at most 2.6 to 6.1 points and Gemma-4-E4B gains on 4 of 5 benchmarks (though its +25.6-point MMLU-STEM gain is flagged as an answer-extraction artifact from a 0.492 L0 parse rate; Appendix G.1), but Claude Haiku 4.5 loses up to 48.0 points (ARC-Easy) and DeepSeek-R1 50.4 points (BoolQ). The L4 15-token budget drops GSM8K accuracy by 27.7 to 82.4 points across the API models. While the paper omits per-model Condition B cost tables for L2-L4, Figure 3 reports four-API-model mean cost changes vs. L0 across all compressed levels (L1 -19% to -41%, L2 -21% to -47%, L3 -19% to -47%, L4 -59% to -83%): L2 and L3 save only 0 to 8 percentage points more than L1 (with L3 slightly less cheap than L2 on 3 of 5 benchmarks), while L4's steeper drop comes from collapsing responses to raw answers.

## Reasoning-Model Caveats (Appendix E and Appendix A.6)

- **Hidden reasoning stays billed:** Compression shortens visible text, not hidden reasoning tokens. On GPT-5.4 at L0 Condition B, billed output averages 2.9x visible output across the five benchmarks (3.5x on BoolQ), so under output compression total priced tokens stay roughly flat while the L1 system prompt adds about 105 input tokens.
- **Smaller open-weight reasoning-group savings:** Under projected public-tier pricing, DeepSeek-R1-Distill (1.17x) and Qwen3.5-9B (1.18x; run with optional thinking mode off in Section 3.6, though Appendix A.6 groups both as "reasoning-distilled") saved less than Gemma-4-E4B (2.09x) and Qwen2.5-VL-7B (2.5x), because their unconstrained L0 generations are already short.
- **Empty visible output under a decoder cap:** Kimi-K2.6 was instrumented but excluded from the eight-model panel because constrained-output runs (L1 and stricter) returned empty visible text on 99-100% of items: its reasoning block consumed the entire per-level `max_new_tokens` budget.

## Scope Caveats (Finding 1, Appendix A.7, Sections 5 and 6.1)

- **Partly mechanical savings:** Savings are largest where the answer is already short (BoolQ yes/no, multiple-choice letters), because L1 collapses the reply to roughly the answer plus minimal reasoning. The paper reads its 1.4x-2.4x range against this format-collapse component; GSM8K, which needs multi-step arithmetic, saves less.
- **Not a truncation effect:** Only 0.9% (GPT-4o), 7.3% (Haiku 4.5), and 0.9% (Qwen2.5-VL-7B) of L1 outputs hit the 300-token ceiling; the paper attributes the L1 savings to natural stopping, not truncation.
- **Text consumers are affected:** The paper notes that surface-text divergence does not matter when only the final answer is used, but does matter for deployments that consume the generation as text (transcripts, audit trails, reasoning displays). Chat explanations fall in that group.
- **Single-turn, short-answer benchmarks only:** All five benchmarks are single-turn QA items with short, structured answers (Sections 3.1, 3.5, 6.1); the paper makes no claim about content preservation in long-form generation or multi-turn chats where prior compressed assistant replies remain in context on subsequent turns.

## Implications for `token-saver`

- Compress only direct assistant chat replies at L1 (Condition B); avoid L2-L4.
- Keep edited files, plans, subagent prompts and reports, and human deliverables in full prose so downstream models never receive Condition A compressed input.
- Adapt Condition B L1 for coding chat (`references/l1-telegraphic.md`): allow negation (`not`, `no`, `never`) and modifiers (consistent with Condition A's POS filter keeping `RB*` and `JJ*`) to avoid inverting technical meaning, ask for reasoning steps only when needed rather than padding simple replies, keep code/paths/commands verbatim, and require a complete, visible reply that never omits content (note: a prompt rule cannot override a tiny host `max_new_tokens` cap like Kimi-K2.6's 300-token ceiling).
- Reset explicitly to full, grammatical English sentences when disabling the skill, omitting Condition B L0's `"Reason step by step"` benchmark CoT directive so normal session verbosity resumes.
- Treat the paper's model numbers as historical data points and measure realized token savings on the specific reasoning models in use.
- Our inference, not a paper result: forbidding content omission and keeping code/commands verbatim bounds savings closer to closed-class word removal (~15-22% of prose tokens in Tables 3 and 8 and Appendix D's `tau = 0.8`) rather than short-answer format collapse, and multi-turn history effects are untested in the paper, so verify net cost and quality empirically.
