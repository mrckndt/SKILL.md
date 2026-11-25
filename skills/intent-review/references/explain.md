# Walkthrough Playbook (`explain` Mode and `--explain`)

Explain how changed code, a module, a PR, or an algorithm works at runtime, for the reader described in the Reader Profile of `SKILL.md`: comfortable with algorithms, complexity, and system concepts, not fluent in reading code or framework conventions. Focus on how data and control move, what gets decided where, and how it scales; skip syntax, types, and boilerplate.

## Step 1: Pick the Scenario

1. With `review --explain`, trace the main changed flow from the review without asking.
2. With `explain [target]`, take the target from the argument (file, function, workflow, algorithm, PR, or the current branch diff). Ask **one** question (Ask tool) only when several unrelated flows exist and the target does not say which one to trace.

## Step 2: Trace From Source

Read the files on disk:

1. **Runtime order**: Follow execution from trigger to outcome (`Trigger → Entry point → Key decisions → State or I/O side effects → Outcome`), not file by file.
2. **Two paths**: Trace one normal (happy-path) scenario and one blocked, boundary, or failure scenario through the real functions.
3. **Scaling**: Count loop nesting, collection scans, recursion depth, retained memory, lock scope, and database or network calls per item. State the result in plain-text Big-O (O(1) lookup, O(n) single pass, O(n log n) sort, O(n^2) nested scan, N+1 queries) and say what `n` counts.
4. **Excerpts**: Quote code at most twice per walkthrough, 3 to 12 verbatim lines each, only at the most important decision points, each followed by what it decides and why. Describe every other stage in plain steps or short pseudo-code.
5. **Discrepancies**: Note any bug, scaling trap, or mismatch between what docs or commits claim and what the code does.

## Writing Rules

1. **Flow before syntax**: Describe data movement, branching rules, state changes, trade-offs, and failure handling.
2. **Framework mechanisms**: The first time a language or framework mechanism affects behavior (goroutine, middleware chain, closure, React effect hook, database transaction, async queue, ORM lazy loading), say in one clause what it does at runtime.
3. **Locations**: Cite code as `` `path/file.ext` (`symbolName`) `` and config or doc files as bare `path/file.ext`.

## Step 3: Emit the Walkthrough

````markdown
## In plain words
<3 to 5 sentences: what this code is for, what happens when it runs, and what a user or caller sees.>

## 1. Mental model
- **Before** (or the problem it solves): <1 to 2 sentences>
- **Now**: <1 to 2 sentences on how requests or data move and what the outcome is>
- **Key files**:
  - `path/file.ext`: <role in this flow>

## 2. Runtime flow
*Normal case: `<scenario>` · Failure case: `<scenario>`*

```text
<small ASCII diagram of the stages, with the failure branch>
```

1. **<Stage name>** — `path/file.ext` (`symbolName`)
   - **What happens**: <plain explanation; define a framework mechanism on first use>
   - **Decision and next step**: <what happens for valid vs. blocked or failing input, and where control goes next>
   - *(At most two stages in the whole walkthrough add a short verbatim excerpt here.)*

*(4 to 8 stages from entry to outcome.)*

## 3. Algorithm, scaling, and design
- **Decision rules or state flow**: <numbered rules, a state table, or a small ASCII diagram>
- **Scaling**: <for example O(n) over <what n counts>; memory; database or network calls per request; flag O(n^2) or N+1>
- **Why this design**: <why this structure, and what trade-off it accepts>
- **Claims vs. code**: Matches | Discrepancy: <exact difference>

## 4. Where it can break
- **Fragile spots**: <inputs, timing races, partial failures, or config combinations that misbehave>
- **How failures show up**: <error messages, status codes, logs, metrics, or "fails silently">

## 5. Check your understanding
- <2 or 3 "What happens when…?" questions with short answers>
- *Name any stage, helper, or failure case above to trace it line by line.*
````
