---
name: c-to-rust-translation
description: Translates a C project into pure Rust with byte-identical behavior, using context-window-aware planning, a persistent PLAN.md that survives context compaction, and aggressive sub-agent delegation. Use when translating, porting, or rewriting C source code, C files, or C libraries into Rust, especially codebases too large to read in a single context window.
license: MIT
metadata:
  author: HARVEST Developers
  version: "1.0.0"
---

# C-to-Rust Translation

Translate the C code into Rust that produces **byte-identical output** for the
same inputs.

Setup assumptions (adapt paths to the actual project):

- The C source lives in a directory of its own — examples below call it
  `c_src/`. **Never modify anything inside it.**
- The Rust Cargo project (Cargo.toml, src/) is created in the current working
  directory, NOT inside the C source directory.
- After translation is complete, the companion `c-to-rust-verification` skill
  (if installed) owns execution-based correctness checking.

## Step 0: Know yourself and plan for context limits

You are a model with a **finite context window**. When your context approaches
its limit, the runtime will automatically **compact** older turns into a lossy
summary. After a compaction:

- You retain a high-level idea of what you were doing
- You **lose** exact file contents, exact function signatures, and the
  fine-grained reasoning that translation depends on
- Re-reading the same files burns more context and can trigger another
  compaction

**Translation is uniquely sensitive to compaction.** Unlike bug-fixing,
translation requires line-level fidelity to the source. A summary like "I read
hash.c (140 lines)" is useless — you need the actual code to translate it.
Therefore: **a single function/module's translation must be completed between
two compactions**, or it will degrade.

Before doing anything else, you MUST:

### 0.1 Self-assessment

Output the following in your first response:

```
Self-assessment:
- Model: <your model name as best you know it>
- Approximate usable context window: <e.g. 200k tokens>
- Approximate budget rule of thumb: ~4 chars per token; 200k tokens ≈ 800kB of text
- Approximate single-response output token limit: <e.g. 16k tokens>
  (If you do not know your output limit, use a conservative estimate of 16k.
   This limit caps how much Rust code a single sub-agent call can write.)
```

### 0.2 Cheap codebase reconnaissance (NO bulk reading)

Estimate the codebase size **without reading file contents**. Prefer cheap
shell-level operations:

```
ls -la c_src/                                  # what files exist
find c_src/ -name '*.c' -o -name '*.h' | xargs wc -l    # line counts
du -sh c_src/                                  # total size
```

From line counts and file sizes, decide which regime you are in:

- **Small (total source < 30% of your window)**: safe to read everything once
  and translate top-down. Skip the rest of Step 0 and proceed to Step 1.
- **Large (> 30% of window)**: full read is impossible. You MUST segment before
  reading anything substantial. Use targeted exploration (Step 0.3) to build a
  picture of the codebase from outlines, not full contents.

Write your decision into your first response so it is auditable.

### 0.3 Lightweight exploration tools — use these instead of reading whole files

Prefer cheap, surgical tools over opening entire files when you only need a
fact, a name, or a signature:

- **`grep` / `rg`**: search for `struct foo`, `typedef`, `^[a-z_]+(`,
  `#define`, function definitions, etc. Scope it
  (`grep -rn 'struct foo_ctx' c_src/`).
- **`head` / `tail` / `sed -n 'A,Bp'`**: read just the first 50 lines of a
  header to see the public API; read just lines 120–180 of a `.c` file to see
  one function.
- **`ls`, `find`, `wc -l`, `du`**: file inventory, sizes, line counts.
- **Small C experiments**: when unsure what a macro expands to or what a C
  idiom actually computes, compile and run a minimal C snippet in a temp
  directory and observe the ground truth, instead of reasoning from first
  principles.

Strategy: **build a high-level mental map first** (what files, what symbols,
who calls whom), then read full contents only for the module you are about to
translate. This keeps each translation subtask within a single uncompacted
window.

### 0.4 Write a persistent plan to `PLAN.md` BEFORE context fills up

If you decided you are in the Large regime, immediately create a file `PLAN.md`
in the current directory (the Rust project root, NOT inside the C source
directory). This file is your **lifeline across compactions**: future-you,
after a compaction, will re-read this file to recover state. It is **not** a
tool's TODO list — it is a plain markdown file you maintain yourself, so it
survives any amount of context loss.

The full template is in **`references/plan-template.md`** — read it now and
create `PLAN.md` from it. PLAN.md must contain (and you must keep up to date):

- **Self-assessment** — model, window, output limit, regime (from Step 0.1)
- **Invariants** — copy the entire `## Invariants` section from the template
  **byte-for-byte**. Do not paraphrase, omit, or reorder. Reason: anything
  outside PLAN.md drifts after compaction; only PLAN.md content reliably comes
  back, so the rules must be byte-stable across the whole run.
- **Codebase summary** — one line per .c/.h with line count and role; project
  type; build configurability; public API surface
- **Translation subtasks** — checkboxed list; sizing constraints below
- **Notes for future-me** — decisions made, pitfalls noticed, where you stopped

**Subtask sizing.** Each subtask must satisfy TWO constraints:

1. **Context window**: the subtask's combined input (C source to read) +
   output (Rust code to write) + tool overhead must fit within ONE uncompacted
   context window. A safe rule: not more than **30%** of your usable window.
   If it would exceed that, split the subtask further.
2. **Output token limit**: the Rust code a sub-agent writes in a single
   response must fit within the single-response output token limit (Step 0.1).
   **Any C file or section exceeding ~1000 lines is very likely to exceed the
   output limit.** Two strategies:
   - **Preferred**: split at the plan level — assign different function groups
     or line ranges of the same file to different subtasks/sub-agents.
   - **Fallback**: instruct the sub-agent explicitly to write the Rust file in
     multiple smaller Write calls rather than one giant write.

Subtask boundaries do NOT need to align with file boundaries. Use the **call
graph** to decide boundaries: group functions that call each other into the
same subtask; split at natural call-graph boundaries where cross-module
dependencies are minimal. A subtask is something future-you (after compaction)
can pick up by reading just PLAN.md plus the listed C files/functions.

If the project includes a test harness entry point that is not part of the
original library, plan to translate it early.

**Rules of engagement with `PLAN.md`:**

1. **Write it BEFORE your context fills up.** It must exist before the first
   compaction. If you wait, it will be too late.
2. **Update the checkboxes and "Notes for future-me" IMMEDIATELY after any
   work completes** — whether it was you or a sub-agent. Do NOT batch updates.
   Compaction can hit at any moment; every second of unrecorded progress may
   lead to work being redone.
3. **After every compaction, re-read `PLAN.md` first thing.** Re-read the
   `## Invariants` section in particular and confirm none of your recent
   actions violated it. Then resume from the first unchecked subtask. Do not
   reconstruct state from memory; trust the file.
4. **Delegate aggressively to sub-agents. Your context window is the
   bottleneck of this whole run — protect it.** Your job as the main agent is
   to OWN the plan and OWN compilation (`cargo build`, error triage,
   feature-matrix verification). Almost everything else — reading C source
   files in detail, writing the corresponding Rust modules, debugging a single
   backend, translating a self-contained primitive — should go to a sub-agent
   so the C code and the new Rust code never have to live in YOUR context.
   Default to delegating; only do a subtask in-process when it genuinely
   depends on shared state you already hold.

   Things you keep:
   - PLAN.md ownership (sub-agents do NOT edit PLAN.md)
   - Cargo.toml / feature-gate decisions
   - Running `cargo build` and routing the resulting errors
   - The cross-module type/ABI design

   When you delegate to a sub-agent:
   - Each sub-agent must report back what files it created/modified and any
     pitfalls it noticed.
   - Update PLAN.md checkboxes and "Notes for future-me" after the sub-agent
     returns, not before.
   - Size the subtask to fit within the sub-agent's output token limit
     (estimate: Rust output ≈ C lines × 1.2, at ~10 tokens/line). A sub-agent
     that hits the output cap mid-write produces an incomplete file and wastes
     the entire run. If a sub-agent returns with truncated output, treat it as
     a signal that the task was too large — split it into smaller pieces, do
     NOT retry the same task at the same size.
   - Pre-inject dependencies into the sub-agent prompt: include the type
     definitions and function signatures it will need from other modules, or
     give it specific `grep` commands, rather than letting it read entire
     files. Every sub-agent that independently reads a 500-line infrastructure
     file wastes thousands of tokens on redundant I/O.

   Rule of thumb: if a subtask would require reading more than ~200 lines of C
   into your own context, delegate it.

### 0.5 Token budget estimation per subtask

For each subtask in `PLAN.md`, do a back-of-envelope estimate before starting:

- Input: total bytes of C files this subtask needs, divided by ~4 for tokens.
- Output: rough estimate of Rust lines you'll write × ~10 tokens/line.
- Tool overhead: each grep/ls/build-error round-trip costs a few hundred to a
  few thousand tokens.

Two independent triggers require splitting a subtask further:

1. **Context window trigger**: estimated total exceeds ~50% of your remaining
   usable window.
2. **Output token limit trigger**: estimated Rust output alone exceeds your
   single-response output token limit.

Either trigger alone forces a split. Better to add three subtasks than to be
compacted mid-write or hit the output cap mid-file.

## Step 1: Analyze BEFORE writing any code

For the subtask you are about to start (or the whole project in the Small
regime):

1. Read only the C files this subtask actually needs. For headers, prefer
   reading just the public-API portion unless you need the macros below.
2. Read the build files (`CMakeLists.txt`, `Makefile`, `configure.ac`, …) to
   understand source file selection and build-time configurability (cache
   variables, options, conditional compilation).
3. Pay attention to preprocessor macros that RENAME functions across the
   project (e.g. `#define foo NAMESPACE(foo)`). These affect the linker symbol
   you will emit in Rust.
4. Determine the project type (record it in `PLAN.md`):
   - Has `main()` → needs a `[[bin]]` target
   - Exports library functions → needs `[lib]` with
     `crate-type = ["cdylib"]` (add `"rlib"` if Rust callers also matter)
   - Both → include both sections
5. Identify ALL backends/variants if the project has build-time
   configurability (project-wide; do it once, in Step 0).

## Step 2: Plan the translation

If the project has build-time configurability (build options selecting
different source files or parameters), you MUST preserve it using Cargo
features. Plan which source files map to which features, and which subtasks in
`PLAN.md` own each feature gate, before writing code.

The feature-naming convention lives in the `## Invariants` section of your
`PLAN.md`. Do not restate the rule from memory — re-read it from PLAN.md
whenever you touch `[features]` in Cargo.toml.

For large projects, break the work into phases: shared/core code first, then
each backend or variant, then wire up feature gates. These phases should
already be the subtasks in your `PLAN.md` — do not re-plan here, just execute
the next unchecked subtask.

## Step 3: Translate

Translate according to `PLAN.md`, preferably with multiple sub-agents for
parallelizable tasks. After each subtask completes:

1. Mark the subtask `[x]` in `PLAN.md`.
2. Append any relevant decision/pitfall to "Notes for future-me" in `PLAN.md`.
3. Then start more subtasks.

The translation rules (C ABI, behavioral fidelity, crate constraints, C-source
boundary) are in the `## Invariants` section of `PLAN.md`. They are the
authoritative source — if unsure about a rule, `cat PLAN.md` and re-read
Invariants. Do not work from memory of this document: this document drifts
after compaction; PLAN.md does not.

### Recovery protocol (if you suspect you were just compacted)

Symptoms: you cannot recall what you just did, or your last turn looks like a
summary rather than concrete work. In that case:

1. `cat PLAN.md` first thing.
2. Re-read the `## Invariants` section. Confirm your most recent code touches
   did not violate any invariant (especially feature naming and C ABI).
3. Find the first unchecked subtask. That is your current work item.
4. Read only the C files that subtask requires (per `PLAN.md`).
5. Resume from there. Do not redo subtasks already marked `[x]`.

## Step 4: Compile check

Run `cargo build --release` and fix any errors until it compiles. If the
project has Cargo features, verify ALL feature combinations compile:
`cargo build --release --features <combo>` for each one.

Once the build is green and all subtasks are checked, mark the whole plan
complete in `PLAN.md` with a final note.

Your job ends when every feature combo's `cargo build` is green. Execution-
based correctness checking (differential testing against the C code) belongs
to the verification pass — see the companion `c-to-rust-verification` skill.
Doing that work here wastes turns. Stop at green compile.
