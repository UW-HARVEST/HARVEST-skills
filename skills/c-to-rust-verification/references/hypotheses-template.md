# HYPOTHESES.md template

Create `HYPOTHESES.md` in the Rust project root from the template below,
right after reading `PLAN.md`. **Copy the `## Invariants` section
byte-for-byte** — do not paraphrase, omit a rule, or reorder. The hypothesis
log section you fill in as you work.

```markdown
# Verification Hypotheses Log

This is an append-only log of bug hypotheses I form while verifying the
Rust translation. After every compaction I will `cat HYPOTHESES.md` first
thing to recover state.

## Invariants (do not drift across compactions)

These rules govern verification. They must survive every compaction unchanged.

### AFTER ANY COMPACTION: `cat PLAN.md HYPOTHESES.md` is your FIRST action before anything.

### Ground truth
- The C code is the authoritative reference. Rust outputs must match C
  byte-for-byte (binary stdout AND every public function output).
- If C and Rust diverge, fix Rust. NEVER modify C.

### Rust toolchain
- Determine the required Rust toolchain up front: `rust-toolchain.toml` in the
  workspace, CI configuration, or the user's stated requirement; otherwise the
  active stable `rustc --version`.
- All builds and tests MUST use that exact toolchain. If a different version
  is active mid-run, stop and report an environment problem instead of
  treating test failures as translation bugs.

### Cargo features (naming contract)
- If the C project has build-time configurability, the Rust crate must expose
  it as Cargo features. Default naming convention: the bare lowercase VALUE of
  each build option (e.g. CMake cache `OPT_A=foo` → feature `foo`). If the
  user or an external test harness specifies expected feature names, that
  specification wins — fix Cargo.toml to expose those names (as primary names
  or as aliases pointing to internal gates), do not change the test command.
- `[lib] name` and `[[bin]] name` MUST use underscores only — NO hyphens.
  Hyphens cause manifest parse failure. Fix Cargo.toml if you see them.

### Boundaries
- Do NOT modify anything in the C source directory.

### Configuration coverage
- Discover every build configuration the project supports (CMake cache
  variables / presets, Makefile options, feature matrices). Every
  configuration must be checked. For each one:
    1. Clean and rebuild C with that configuration's build flags.
    2. Rebuild Rust with the matching Cargo features
       (`cargo build --release --no-default-features --features <list>`).
    3. Re-run integration tests and fix any mismatches before moving on.

### Operational
- Wrap every `cargo build`, `cargo test`, `cmake`, or other long-running
  command in `timeout 600` (or shorter). No single command should run > 600s.
- If a single test takes too long, skip it and move on. Do not get stuck on
  one step.

## Hypothesis log

Format per entry:
### H<N>: <one-line hypothesis>
- Status: open | confirmed | refuted | fixed
- Evidence: <how I think I know>
- Files/lines suspected: <file path:line>
- Action taken: <Edit/test/none yet>
- Outcome: <what happened after the action>
```
