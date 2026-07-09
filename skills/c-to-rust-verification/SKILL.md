---
name: c-to-rust-verification
description: Verifies and fixes a C-to-Rust translation by differential testing against the original C as ground truth — builds the C as a shared library, compares C and Rust outputs byte-for-byte with libloading-based tests, tracks bug hypotheses in a persistent HYPOTHESES.md that survives context compaction, and delegates fixes to sub-agents. Use after translating C code to Rust, or when asked to verify, test, debug, or fix a Rust port of a C library or program.
license: MIT
metadata:
  author: HARVEST Developers
  version: "1.0.0"
---

# C-to-Rust Verification

You are testing a C-to-Rust translation for correctness. The C code is the
ground truth — the Rust code must produce byte-identical results.

Setup assumptions (adapt paths to the actual project):

- The original C source lives in its own directory — examples below call it
  `c_src/`. **Never modify anything inside it.**
- The Rust translation lives in the current working directory (`Cargo.toml`,
  `src/`).
- The C code can be compiled as a shared library to serve as the oracle. Look
  at its build files (e.g. `c_src/CMakeLists.txt`) to understand the build
  system. For CMake projects:

  ```
  cd c_src && mkdir -p build && cd build && \
  cmake .. -DCMAKE_POSITION_INDEPENDENT_CODE=ON <project-specific -D flags> && \
  cmake --build .
  ```

  Then find the resulting `.so` files in the build output.

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

Now do the actual verification:

1. Build the C code as a shared library.
2. Write Rust integration tests (in `tests/`) that use `libloading` to load
   the C `.so` and compare C vs Rust function outputs.
3. Start with the lowest-level functions and work upward to higher-level ones.
   Look at the C headers to identify the public API and call hierarchy.
4. For each function: create fixed test inputs, call both C and Rust versions,
   assert outputs match byte-for-byte.
5. Run `cargo test` and investigate any mismatches. Every time a test exposes
   a divergence, append a hypothesis to `HYPOTHESES.md`.
6. When you find a Rust function that produces different output than C, fix
   the Rust code in `src/` and re-run until the test passes. Update the
   matching hypothesis to `fixed` after the Edit.
7. Keep going until all public functions match.
8. If the project has a main binary, run both the C binary and the Rust binary
   with the same inputs and compare their stdout byte-for-byte. Fix any
   differences.
9. Compare `nm -D` on the C `.so` and the Rust `.so`. Every symbol the C `.so`
   exports, the Rust `.so` must also export with the exact same name — this
   includes symbols created by preprocessor macros. No exceptions. Add missing
   exports.

When a divergence hinges on subtle C semantics (macro expansion, integer
promotion, undefined-behavior-adjacent idioms), compile and run a minimal C
snippet in a temp directory and observe the ground truth directly, instead of
reasoning from first principles.

All operational rules (libloading dev-dependency, the C-source boundary,
per-configuration re-verification, the 600-second timeout cap) live in the
`## Invariants` section of your `HYPOTHESES.md`. Re-read them from
`HYPOTHESES.md` whenever you are unsure — do not work from memory of this
document.
