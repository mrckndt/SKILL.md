# Evidence

What the rules are based on, and what the sources do not show. Rules in `SKILL.md` carry no labels. This file carries the scope.

Primary source: `2606.24083v1.pdf`, CAVEWOMAN, Adeyemi, Dernoncourt, Rossi, June 2026. https://arxiv.org/abs/2606.24083

## What the paper measured

Eight models (four API, four open-weight), five short-answer datasets (GSM8K, BoolQ, ARC-Easy, CommonsenseQA, MMLU-STEM), five reduction levels, greedy decoding, English. Two channels: input compression (the prompt is filtered before the model sees it) and output compression (the system prompt constrains the response).

The L1 output prompt, verbatim (Appendix H.2):

> Answer the question under a TELEGRAPHIC constraint.
>
> Rules:
> - DO NOT use any function words. No articles (the, a, an). No conjunctions (and, but, or, so). No prepositions (of, in, to, for, at, with, from, by, on, per).
> - DO use nouns, main verbs, numbers, and standard symbols (+, -, *, /, =).
> - Show each reasoning step.
> - End with a line: 'Answer: <answer>'.

The POS filter (Condition A, §3.2) drops DT, IN, CC, RP, TO, and MD. It keeps every other tag, including adjectives, adverbs, and verbs. The filter is not the same thing as the output prompt, and the filter can drop words the meaning needs. That is why `SKILL.md` has a meaning floor above the register.

## Why one register

The paper's NLI judge is reliable at L1 and degrades above it (Appendix C.1, false-negative rate on synthetic positive pairs):

| Level | Judge false negative |
|---|---|
| L1 | 2.9% |
| L2 | 28.6% |
| L3 | 50.0% |
| L4 | excluded from semantic evaluation |

Only L1 has a semantic result worth trusting. That is the reason this skill implements one register, not a preference.

## Claims

| Claim | Value | Unit | Panel / condition | Source | Scope |
|---|---|---|---|---|---|
| Output compression cost | 1.4 to 2.4x cheaper per model, up to 3x best cell | realized cost multiple | L1-B, GPT-4o, Haiku 4.5, Sonnet 4.6; 17 of 20 API cells | §4.1 | short structured answers; format collapse; not a token percent; GPT-5.4 flat (2 of 5) |
| Surface divergence | 51.9% | share of generations | six non-reasoning models, L1-B | §4.2 | correct but non-entailing; 80.4% length-controlled; 42.7% on eight models |
| Input compression net cost | +15% mean, up to 1.8x on a dataset at L1, about 2.7x at deeper levels | cost multiple | API panel | §4.1, Table 3 | input tokens fall; net rises through longer responses |
| Instruction overhead | about 105 | input tokens | L1-B system prompt | Appendix E | per call; this skill's text adds its own |
| Hidden reasoning tokens | billed output 2.9x visible on average | ratio | GPT-5.4, five benchmarks | Appendix E | visible text shrinks, bill does not; 2 of 5 benchmarks cheaper |
| Open-weight savings | 2.5x Qwen2.5-VL, 2.09x Gemma-4, 1.18x Qwen3.5, 1.17x DeepSeek-R1 | projected cost multiple | public price panel | Appendix A.6 | projected, not metered |
| Agentic output | -8.5% | percent of output tokens | 82 paired tasks, Claude Code, claude-sonnet-5 low effort, forced activation | JetBrains | ceiling, not typical; p=0.82; raw cost +11.6% from one outlier |
| Elastic token reduction | 63.6% | percent of tokens | 8 scenarios, live MCP, word-based count (about 1.3 tokens/word) | Elastic Labs | not a model A/B; EVAL arms are simulated wrappers; 35% to 92% by payload |
| Caveman skill vs "Answer concisely" | 3% median (caveman), 35% (ultracave), 9% (megacave) | percent of output tokens | 10 prompts, single run, claude-opus-5-5, skill text 3.1.0 | `caveman/docs/HONEST-NUMBERS.md` | n=10, output length only, no quality claim |

## Provenance of the rules

| Rule | Origin | Note |
|---|---|---|
| 1. Floor: payload and meaning exact | Engineering rule plus paper-informed overlay | The L1 filter drops determiners, so it removes the determiner negator "no" and DT/PDT quantifiers ("all", "some", "each"); "not" is an adverb and survives the filter and the H.2 prompt. The floor is an addition, not a paper finding |
| 2. Answer placement | Owner interface decision; `Answer:` line from Appendix H.2 | Coding: result-first is an adaptation, not validated. Single-value questions keep the paper's steps-then-`Answer:` order, since the answer-last line is an extraction format |
| 3. Telegraphic register | §3.2 and Appendix H.2 | The one measured output register |
| 4. Kill ceremony | Engineering rule | The paper does not test greetings or closers |
| 5. One idea per sentence | Engineering rule | Not measured |
| 6. Tool runs: bounded status | Engineering rule | Not measured; the paper has no tools |
| 7. No caricature | Readability rule | The paper neither tests nor requires abbreviations or arrows |
| 8. Language and layering | Engineering rule | The paper tests English only; layering is not measured |

## Paper scope limits

Quoted from §5 and §6.1:

- "we make no claim about content preservation in long-form generation tasks such as summarization or open-ended QA."
- "The accuracy/reference-text divergence is a surface-text observation, not a propositional claim."
- "two API vendors at two models each; we therefore draw no vendor- or family-level conclusion."
- All five benchmarks have short, structured answers (numeric, boolean, multiple choice).
- Greedy decoding only. Single-turn. No tools, no code, no repo work.
- 45 of 80 model/dataset/condition cells have L0 extraction rate below 0.95. Accuracy gains are reported only where the L0 rate is at least 0.95.
- The L0-A versus L0-B system-prompt noise floor is not separately measured.

## Secondary sources

**JetBrains** (`Speaking to AI Agents like Cavemen Saves 65_ of Tokens. We Test.html`): Harbor 0.17, Claude Code 2.1.200, claude-sonnet-5 with effort low, SkillsBench 86 of 87 tasks, 3 runs, about 240 trials, about USD 106. Forced activation. Output fell 8.5% (592k to 542k over 82 paired tasks). Per-task scores 0.326 versus 0.311, sign test p=0.82 (8 better, 10 worse, 64 tied). Raw arm cost was 11.6% higher because one long-context trial billed USD 8.29 versus USD 0.33. Limit: forced activation is the ceiling; "no detectable difference" is not equivalence.

**Elastic** (`Elastic-caveman for token reduction with Claude _ Elasticsearch Labs.html`, `elastic-caveman/EVAL.md`): 8 scenarios against a live Elastic MCP endpoint. 1,284 normal tokens versus 467 caveman tokens, 63.6% average. Metadata listings 85% to 92%, ES|QL generation 70% to 75%, data-heavy searches 35% to 40%. The blog reports 88% success while EVAL.md reports 8 of 8. EVAL arms are simulated wrappers, and token counts are word-based. Limit: use the preservation checklist (field names, IDs, nulls, error strings, API paths) and the payload pattern, not the percentage as a model A/B.

**Caveman repo** (`caveman/docs/HONEST-NUMBERS.md`, `caveman/evals/README.md`): against a plain "Answer concisely." control, the skill adds 3% median output reduction (n=10, single run). LemonCrow: output -44.1%, total cost +0.06%, initial cached context +1,532 tokens. edubraqd: across 662 Claude Code sessions, output was 0.3% of all tokens and net effect on the bill was within plus or minus 0.2%. Limit: different skill text and versions; the incremental effect over "be concise" is the honest comparison.

## Which source answers which question

| Question | Source |
|---|---|
| Does compressing the response cut cost? | Paper, short-answer panel |
| Which register? | Paper, L1 only |
| Does it help a coding agent? | JetBrains, single vendor, forced activation |
| What must stay exact in data answers? | Elastic checklist |
| Does it beat "be concise"? | Caveman eval, n=10 |
| Does this skill help you? | Not measured |

## Do not claim

- 65% or 75% savings. Those are old caveman numbers for chat prose, not agent work.
- That compressing prompts or pasted files saves money. Input tokens fall but net cost rises.
- That a cost multiple is a token reduction. 1.4 to 2.4x is realized cost at L1-B, not tokens.
- Quality equivalence, "zero loss", or "no quality loss". JetBrains found no detectable difference.
- Content preservation in long-form writing. The paper disclaims it.
- Savings on per-request or credit billing.
- Savings when hidden reasoning tokens are billed.
- Paper support for coding, tools, answer-first, or non-English use.
- That 51.9% means half the answers are wrong. It means correct answers whose surface text diverges.
- That Elastic's 63.6% is a model A/B or a general result.
- That open-weight savings are metered dollars. They are projected.
- Any percentage in `SKILL.md`.

## Manual A/B

- [ ] Three arms: no instruction, `Answer concisely.`, this skill active on every reply.
- [ ] Same model, harness, repo state, budget, and task set. Paired runs.
- [ ] At least 3 runs per arm. Do not cite a k=1 result.
- [ ] Measure provider-billed input, output, cache-read, cache-write, and total cost.
- [ ] Report output-token change separately from total-cost change, including the skill's own input overhead.
- [ ] Grade with the task's tests or a pre-registered rubric.
- [ ] Pre-register the decision rule: keep only if total billed cost is lower and quality is not worse, or the drop is acceptable.
- [ ] Report outliers and failures. One long-context task can invert the cost result.
- [ ] Record model, harness, skill version, date, and activation mode.

## Open questions

- Does answer-first change accuracy or comprehension against the paper's answer-last order?
- Does L1 help or hurt coding accuracy on real repos?
- Does the skill's input overhead wash out output savings in agent harnesses with cache reads?
- Does L1 transfer to languages other than English?
- Does L1 preserve content in long-form writing?
