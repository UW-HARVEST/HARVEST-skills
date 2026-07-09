# PLAN.md template

Create `PLAN.md` in the Rust project root from the template below. Fill in the
Self-assessment, Codebase summary, Translation subtasks, and Notes sections
from your analysis. **Copy the `## Invariants` section byte-for-byte** — do not
paraphrase, omit a rule, or reorder. Invariants must be byte-stable across the
whole run so they survive every compaction unchanged.

```markdown
# Translation Plan

## Self-assessment
- Model: ...
- Window: ...
- Output token limit: ... (unknown → use 16k as conservative default)
- Regime: small | medium | large

## Invariants (do not drift across compactions)

These rules are not negotiable and must survive every compaction unchanged.
When in doubt, re-read this section.

### AFTER ANY COMPACTION: `cat PLAN.md` is your FIRST action before anything.

### Rust toolchain
- Determine the required Rust toolchain up front: `rust-toolchain.toml` in the
  workspace, CI configuration, or the user's stated requirement; otherwise the
  active stable `rustc --version`.
- All builds and self-tests MUST use that exact toolchain. If a different
  version is active mid-run, stop and report an environment problem instead of
  treating build failures as translation bugs.

### Cargo features
- If the C project has build-time configurability (build options selecting
  different sources or parameters), every configuration must be preserved as a
  Cargo feature. Do NOT hardcode a single configuration.
- Default naming convention: the feature name is the bare lowercase VALUE of
  each build option (e.g. CMake cache `OPT_A=foo`, `OPT_B=bar` → features
  `foo`, `bar`). If the user or an external test harness specifies expected
  feature names, that specification wins — honor it exactly.
  RIGHT — bare values directly:
      [features]
      foo = []
      bar = []
      "2k" = []
  ALSO ACCEPTABLE — prefixed gate + bare alias (useful when Cargo dislikes
  a bare name; the alias keeps the naming contract intact):
      [features]
      opt_c_2k = []
      "2k" = ["opt_c_2k"]
  WRONG — prefixed without an alias to the bare value:
      opt_a_foo = []           # NO — consumers expect `--features foo,bar,2k`
- ALL feature combinations must compile
  (`cargo build --release --features <combo>`).

### Cargo manifest target names
- `[lib] name` and `[[bin]] name` MUST use underscores only — NO hyphens.
  Hyphens in target names cause `cargo` to fail parsing the manifest entirely.
  RIGHT: `name = "sphincs_plus"`, WRONG: `name = "sphincs-plus"`.

### C ABI
- Public C exports use `#[unsafe(no_mangle)]` and `extern "C"` with exact C
  signatures (use `*const c_char`, `c_int`, etc. from `std::ffi`).
- The exported symbol name is the FINAL linker symbol after all preprocessor
  renames. If C has `#define foo NAMESPACE(foo)` producing `PREFIX_foo`, the
  Rust export is named `PREFIX_foo`, not `foo`.

### Behavioral fidelity
- Do NOT fix bugs in the original C. Reproduce behavior exactly.
- Preserve the exact order of error checks and validation.
- Match C's stdin reading semantics (scanf reads across newlines; fgets does not).
- Match C's exact printf format including spacing and newlines.

### Crate constraints
- Do NOT use the `openssl` crate or any OpenSSL bindings. Use pure-Rust crates.
- Prefer safe Rust internally; do not relax the C ABI on exports.

### Boundaries
- Do NOT modify anything in the C source directory.

### Translation fidelity
- You MUST faithfully translate ALL C source files to **pure Rust**. Do NOT
  use the `cc` crate (or any equivalent) in `build.rs` to compile or link the
  original C source code. Assume the C source will NOT exist wherever the
  translated crate is later built or tested — the only code available then is
  the Rust you write. Any attempt to wrap C via a compiled static archive or
  object file defeats the purpose of the translation.
- Do NOT import or depend on any existing Rust crate that implements, wraps,
  re-exports, or compiles the same C library you are translating. Every line
  of Rust code must be written by you. If a function needs to call out to
  system libraries (e.g. POSIX APIs), use `libc` or equivalent thin FFI
  crates, not crates that compile the library you are meant to translate.
- A `build.rs` is allowed for legitimate build-time needs (code generation,
  feature detection, etc.), but it must NOT reference, compile, or link any
  file from the C source directory.
- No shortcuts: every function, every struct, every constant, every macro in
  the C source must become a corresponding Rust implementation. Stub functions
  that return 0 or a hardcoded value are NOT acceptable unless the function's
  return value is defined by the API contract as a compile-time constant.

## Codebase summary
- Files: <one line per .c/.h with line count + 1-line role>
- Project type: bin / lib / both
- Build configurability: <Cargo features needed, if any>
- Public API surface: <list of public functions/types>

## Translation subtasks

<!-- Each subtask must fit ONE uncompacted context window (≤30% of usable
     window for input+output+overhead) AND its Rust output must fit the
     single-response output token limit. Split large C files by function
     group or line range. See the skill's Step 0.4 for sizing rules. -->

- [ ] T1: <name> — files/functions: <list> — estimated output: ~Nk tokens — depends on: <other Tx>
- [ ] T2: ...
- [x] T3: ...   <!-- mark done as you go -->

## Notes for future-me (post-compaction)
- Decisions already made and why
- Cargo features chosen and what they gate
- Pitfalls noticed (e.g. macro renames, namespace prefixes)
- Where you stopped and what to do next
```
