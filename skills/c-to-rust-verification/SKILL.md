---
name: c-to-rust-verification
description: Verifies and fixes a C-to-Rust translation by differential testing against the original C as ground truth — compiles the C into a GoogleTest binary as the oracle, loads the translated Rust as a shared library, compares outputs byte-for-byte with fixed tests and coverage-guided FuzzTest properties, tracks bug hypotheses in a persistent HYPOTHESES.md that survives context compaction, and delegates fixes to sub-agents. Use after translating C code to Rust, or when asked to verify, test, debug, or fix a Rust port of a C library or program.
license: MIT
metadata:
  author: HARVEST Developers
  version: "1.1.0"
---

# C-to-Rust Verification

You are testing a C-to-Rust translation for correctness. The C code is the
ground truth — the Rust code must produce byte-identical results.

Setup assumptions (adapt paths to the actual project):

- The original C source lives in its own directory — examples below call it
  `c_src/`. **Never modify anything inside it.**
- The Rust translation lives in the current working directory (`Cargo.toml`,
  `src/`).

The concrete mechanism you use to compare C against Rust — how you build each
side and where you write the comparison — is described in **Step 2** below.
Everything before it (PLAN.md, the HYPOTHESES.md discipline, the invariants)
applies regardless of which comparison mechanism you use.

## Step 0: Read PLAN.md FIRST

A previous translation agent may have left a file `PLAN.md` in this directory
containing its design notes, parameter tables, decisions, and pitfalls it
noticed during translation (the companion `c-to-rust-translation` skill
produces one). **Before doing anything else**, run:

```
cat PLAN.md
```

If `PLAN.md` exists, treat it as authoritative background. Do NOT re-derive
project structure, module layout, Cargo features, parameter values, or design
rationale from scratch — that information is already there. Pay particular
attention to the "Notes for future-me" section: the translation agent may have
flagged specific concerns (e.g. "macro renames", "padding edge cases") that
point directly at likely bug sites.

If `PLAN.md` does not exist, proceed without it.

## Step 1: Maintain `HYPOTHESES.md`

Verification work involves forming hypotheses about why the C and Rust outputs
differ, then confirming or refuting them. Your context window is finite and
will be **compacted** when it fills up; after a compaction your memory of which
hypotheses you already investigated is **lost**, leading to "rediscover the
same bug three times" loops that waste an entire run.

To prevent this, you MUST maintain a file `HYPOTHESES.md` in the current
directory as an append-only log of bug hypotheses. Create it at the very start
of your work (right after reading `PLAN.md`) from the template in
**`references/hypotheses-template.md`** — read that file now.

**Rules of engagement with `HYPOTHESES.md`:**

1. **The `## Invariants` section is verbatim.** Copy the entire `## Invariants`
   block from the template byte-for-byte. Do not paraphrase, omit, or reorder.
   The hypothesis log section you fill in as you work; Invariants is fixed
   text. Reason: anything outside HYPOTHESES.md drifts after compaction; only
   this file reliably comes back.
2. **Every time you form a new hypothesis** (e.g. "I think `foo()` has an
   off-by-one in the padding length"), append a new `### H<N>` entry
   immediately, with status `open`. Do NOT wait until you have proof.
3. **After running a test that bears on a hypothesis**, update its `Status` to
   `confirmed` or `refuted` and write the evidence in `Outcome`. Do NOT leave
   entries stale.
4. **After applying an Edit that you believe fixes a hypothesis**, mark it
   `fixed` and note what you changed.
5. **Before forming a new hypothesis, check if it is already in the file.** If
   so, do not re-investigate it — read its current Status and proceed.
6. **After every compaction, `cat HYPOTHESES.md` first thing.** Re-read the
   `## Invariants` section, then the hypothesis log. If the very first
   hypothesis you form already exists with status `confirmed` or `fixed`, you
   are in a thrashing loop — stop, re-read the existing entry, and continue
   from where it left off (e.g. if status is `confirmed` but not yet `fixed`,
   your next action is to apply the fix, not re-confirm).
7. The file is for **future-you across your own compactions**, not for
   sub-agents. Do not delegate its upkeep; you maintain it yourself.
8. **Delegate fixing work aggressively. Your context window is the bottleneck
   — protect it.** Your job as the main agent is to OWN HYPOTHESES.md and OWN
   execution: building C and Rust, running tests, running `nm`, comparing
   C-vs-Rust outputs, deciding which functions diverge. Almost everything else
   — reading large C source files to understand an algorithm, locating the
   matching Rust code, applying the actual fix — should go to a sub-agent so
   neither the C nor the buggy Rust ever has to live in YOUR context. Default
   to delegating; only do a fix in-process when it is a one-line change you
   can apply from what you already see.

   Things you keep:
   - HYPOTHESES.md ownership (sub-agents do NOT edit HYPOTHESES.md)
   - Building C / Rust, running cargo test, running nm, output comparison
   - Hypothesis status updates after each test run
   - Per-configuration coverage tracking

   Rule of thumb: if investigating or fixing a hypothesis would require
   reading more than ~200 lines of C or Rust into your own context, delegate
   the fix to a sub-agent and let it report back what it changed.

### Recovery protocol (if you suspect you were just compacted)

Symptoms: you cannot recall what hypothesis you were testing, or your last
turn looks like a summary rather than concrete work. In that case:

1. `cat PLAN.md HYPOTHESES.md` first thing.
2. Find the first hypothesis with status `open` or `confirmed` (but not yet
   `fixed`). That is your current work item.
3. Resume from its `Action taken` field. Do not redo work already logged.

## Step 2: Verification workflow

You compare C against Rust from **inside a C++ GoogleTest binary**. A
ready-to-use test environment ships with this skill in `assets/verify_env/` —
copy that whole directory into the project root as `verify_env/`. The C
reference is compiled directly into the test binary; the translated Rust is
loaded as a shared library and called through its C-ABI exports. Every test
input runs both sides and compares the observable result with GoogleTest
assertions.

The full workflow — the environment's contents, the compile-definitions check
on the C reference, the test-writing steps, and the FuzzTest property-fuzzing
discipline — is in **`references/gtest-method.md`**. Read it now and follow
it.

When a divergence hinges on subtle C semantics (macro expansion, integer
promotion, undefined-behavior-adjacent idioms), compile and run a minimal C
snippet in a temp directory and observe the ground truth directly, instead of
reasoning from first principles.

All operational rules (the C-source boundary, per-configuration
re-verification, the 600-second timeout cap) live in the `## Invariants`
section of your `HYPOTHESES.md`. Re-read them from `HYPOTHESES.md` whenever
you are unsure — do not work from memory of this document.
