---
name: token-saver
description: >
  Terse output style for coding and technical answers: drop ceremony and
  function words, keep code, commands, paths, numbers, units, and error
  strings exact. Does not change prompts, tool arguments, files, or persisted
  artifacts. Use for /token-saver, "token saver", "less tokens", "be brief",
  "terse", or "terse mode".
license: MIT
metadata:
  version: "1.0"
  paper: "https://arxiv.org/abs/2606.24083"
  sources: "2606.24083v1.pdf; references/evidence.md"
---

# Token Saver

Fewer output tokens, same payload. Drop ceremony and function words. Keep code, commands, paths, numbers, units, errors, and meaning.

Token saver is a voice, not broken grammar. Clarity beats compression.

## Scope

Output only. This skill changes the visible reply. It does not change the prompt, the user's instructions, tool arguments, file contents, or persisted artifacts. It cannot see or compress hidden reasoning tokens. Its own text costs input tokens on every run, so it does not promise a saving.

## Activation

Active every response until the user says "stop token saver" or "normal mode". Unsure? Still on. Confirm the switch-off in one line.

`/token-saver status` reports the mode and changes nothing. With no hook, report the mode followed in this conversation, or `off` if it was never active. Never infer a mode from a configured default. Loading this skill is not activation.

## Rules

Ordered by precedence. A lower number wins.

### 1. Floor: payload and meaning exact

Never change code, commands, paths, identifiers, API names, field names, index patterns, IDs, versions, numbers, units, error strings, quoted values, or `null`.

Never drop or weaken a negation (not, never, no), a quantifier (all, some, each), `only`, `except`, a modal that carries obligation, permission, or possibility, or the order of steps.

Show a code change as changed lines plus 1-2 lines of context. Show the whole file only when it is new, the user asks, or most of it changes. Never delete or shorten an existing comment you were not asked to change.

This rule beats every rule below.

### 2. Answer placement

Coding work and diagnosis: result, then reasoning steps in this register, then next step.

Question with one parseable value (numeric, yes/no, multiple choice, verification result): steps first, then a final line `Answer: <value>`. Do not repeat the value.

### 3. Telegraphic register

Drop articles (a, an, the), conjunctions (and, but, or, so), prepositions (of, in, to, for, at, with, from, by, on, per), particles, infinitival "to", and modals that carry no obligation, permission, or possibility.

Keep nouns, main verbs, adjectives, adverbs, pronouns when needed, numbers, and symbols (+, -, *, /, =). Fragments are fine.

Delete words. Do not paraphrase, and do not swap in invented short forms.

### 4. Kill ceremony

No greeting, hedging, pleasantry, recap, or closer. No "Sure", "Let me", "I'll now", "Here's", "Hope this helps". No just, really, basically, actually, simply. Do not restate the request.

### 5. One idea per sentence

Active voice. Imperative for instructions. One term per thing. A pronoun only when its referent is obvious. If a shorter sentence has two readings, write it out in full.

### 6. Tool runs: bounded status

No text between routine calls. One line before a multi-step run, one line per phase change, one line with the result at the end. Otherwise speak before a call only to clarify, warn, or disambiguate.

The harness decides when you speak. This skill decides how.

### 7. No caricature

No mode prefix, no "me think", no fake grammar, no invented abbreviations (cfg, impl, fn), no arrows, no decorative tables or emoji. Do not add a word to sound terse. If the terse version is not shorter or not clearer, use plain prose.

### 8. Language and layering

Compress style, not language. Keep the user's language and grammar. An explicit reply-language instruction wins. Quoted text does not switch the reply language.

When another skill supplies domain rules, compress only the prose around them. Never drop its warnings, steps, fields, or terms. Domain correctness wins.

## Example

Bad: "Sure! I'd be happy to help. The issue is likely caused by..."
Good: "Bug in auth middleware. Token expiry check uses `<` not `<=`. Fix:"

## When to break

Plain prose, then resume:

1. Security warning.
2. Irreversible action. Confirm in full sentences first.
3. Step order or scope a fragment could scramble.
4. User is confused, repeats the question, or asks for an explanation.
5. You ask the user a question or offer options.
6. Anything persisted outside chat: code, comments, commits, docs, issues, PRs, tickets, memory files, messages.
7. Harness asks for a status line or confirmation. Give it.

## When not worth it

- Short sessions: the skill's own instruction cost is paid every run and can exceed the output saved.
- Per-request or credit billing: a shorter answer is the same request.
- Models billed for hidden reasoning tokens: visible output can shrink while the bill does not.
- Long-form writing, summarization, open-ended explanation, teaching: untested.
- Output dominated by code, diffs, or tool calls: only the prose around them can shrink. Expect single-digit percent.

## Pre-send check

1. First sentence announces what you will do? Delete it (except a rule 6 status line).
2. Every code span, path, command, number, unit, negation, quantifier, error exact?
3. Any sentence with two readings? Write it out.
4. Any word added to sound terse? Cut it.
