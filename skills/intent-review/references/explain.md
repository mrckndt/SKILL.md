# Guided Systems & Flow Walkthrough Playbook (`explain` Mode and `--explain`)

Translate implemented changes, modules, PRs, or algorithms into a clear runtime walkthrough calibrated for a technical reader who understands systems, APIs, state, configs, logs, and algorithmic complexity, and wants the core logic, design decisions, and scaling behavior without line-by-line variable or datatype clutter.

## Step 1: Confirm Target Scenario

1. When invoked via `review --explain`, trace the primary changed flow from the active review without re-prompting.
2. When invoked as standalone `explain [target]`, identify the target from the user's argument (specific file, function, workflow, algorithm, PR, or current branch diff). Ask **one** brief question (using `AskQuestion` when available) only if multiple unrelated workflows exist and it is unclear which path to trace.

## Step 2: Trace Runtime Execution and Complexity Directly From Source

Inspect the repository files directly on disk:

1. **Trace chronological runtime order**: Follow execution from entry trigger to final outcome (`Trigger -> Entry Point -> Key Decisions -> State/IO Side Effects -> Outcome`), rather than walking files alphabetically.
2. **Trace two concrete paths**: Follow one **primary (happy-path) scenario** and one **blocked, boundary, or failure scenario** through the actual functions and callers.
3. **Evaluate core algorithms and scaling**: Count loop nesting, collection scans, recursion depth, memory retention, lock contention, and database/network calls per item. Express scaling explicitly (for example \(O(1)\) lookup, \(O(n)\) single pass, \(O(n \log n)\) sort, \(O(n^2)\) / \(O(n^3)\) nested scan, or N+1 I/O queries) and state what input variable `n` represents in plain terms.
4. **Extract short verbatim excerpts**: Read exact 3 to 12 line excerpts from disk for the pivotal decision points only, pairing each snippet with a plain explanation of what it decides and why.
5. **Flag spotted discrepancies**: While standalone `explain` skips Pass A and Pass B subagents, record any concrete bug, scaling trap, or mismatch between commit/spec claims and actual code uncovered during tracing.

## Writing Calibration Rules

1. **Focus on flow and decisions over syntax**: Center prose on data movement, branching rules, state mutations, architectural tradeoffs, and failure handling. Skip routine type annotations, boilerplate imports, and elementary syntax explanations.
2. **Define idioms once in plain systems language**: On the first appearance of a language or framework mechanism that affects behavior (such as a Go goroutine/channel, middleware chain, closure capture, React effect hook, database transaction lock, or async queue), state what runtime job it performs in one short clause.
3. **Keep locations scannable**: Cite code stops as `` `path/file.ext` (`symbolName`) `` (honoring stricter base-repo citation rules when present) and config or doc files as bare `path/file.ext`.

## Step 3: Emit the Guided Logic & Flow Walkthrough

Output the walkthrough using this structure:

````markdown
## 1. Mental Model (System Overview)
- **Before (or Baseline)**: <1 to 2 sentences on how the system behaved prior to this change, or what problem this module solves>
- **Now (Core Mechanism)**: <1 to 2 sentences on how requests/data move through the code and what outcome callers or users observe>
- **Key files & roles**:
  - `path/file.ext`: <one-line responsibility in this flow>

## 2. End-to-End Runtime Flow (Execution Order)
*Tracing primary scenario: `<concrete scenario>` (plus divergence on failure/blocked case: `<blocked/error scenario>`)*

1. **<Stage Name - e.g., Entry & Guard Check>** - `path/file.ext` (`symbolName`)
   - **What happens**: <plain systems explanation of trigger, key inputs, and any framework idiom on first use>
   ```<lang>
   <short verbatim excerpt read from disk showing the pivotal decision>
   ```
   - **Decision & handoff**: <what branch runs on valid vs. blocked/error input, and which function receives control next>

*(Include 4 to 8 key stops covering the full path from entry to completion.)*

## 3. Core Algorithm, Complexity & Design Decisions
- **Decision rules / state flow**:
  - <Numbered rules, state transition table, or compact ASCII flow diagram>
- **Time, memory & I/O scaling**: <explicit complexity such as O(n) over `<what n is>`, memory footprint, and DB/network calls per request; call out any O(n^2), O(n^3), or N+1 behavior>
- **Why this design (and tradeoffs)**: <why the implementation chose this structure over alternatives, and what tradeoff it accepts>
- **Intent vs. code alignment**: Matches stated intent | Discrepancy spotted (<exact difference between claimed behavior and actual code>)

## 4. Edge Cases, Failure Modes & Runtime Visibility
- **Fragile spots & open edge cases**: <inputs, concurrency/timing races, partial failures, or configuration combinations that could misbehave>
- **Runtime diagnostics**: <exact error messages, status codes, metrics, logs, or settings operators/developers can inspect - or note where failure visibility is silent>

## 5. Mental-Model Check & Zoom-In
- <2 to 3 concrete "What happens when..." edge-case prompts with brief answers so the reader can verify their mental model>
- *To zoom deeper, name any step, helper, data structure, or failure scenario above to trace line by line.*
````
