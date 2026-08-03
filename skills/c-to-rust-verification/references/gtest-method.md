# Verification workflow (GoogleTest)

You compare C against Rust from **inside a C++ GoogleTest binary**. A
ready-to-use test environment ships with this skill in `assets/verify_env/` —
copy that whole directory into the project root as `verify_env/` (its
CMakeLists expects the C source at `../c_src/`; adjust if the layout differs).
The C reference is compiled directly into the test binary; the translated Rust
is loaded as a shared library and called through its C-ABI exports. Every test
input runs both sides and compares the observable result with GoogleTest
assertions.

`verify_env/` already handles the infrastructure so you can focus on the API
behavior:

- `CMakeLists.txt` — fetches a pinned GoogleTest and FuzzTest, compiles
  `c_src/` into the test binary, links everything, and registers the tests.
- `verification_tests.cc` — the test file you write and extend.
- `harvest_diff.h` — helpers for differential comparison: `FilledBuffer` (0xA5
  pre-fill to expose short/uninitialized writes), `FirstDifference`/`HexDump`/
  `Explain` (attach `<< harvest::Explain(c, rust)` to an `EXPECT_EQ` to get the
  first differing offset and a hex dump on failure), `WithErrno`, and a
  `Observation` struct to extend. Reuse or extend as the API needs.
- `rust_lib.h` — loads the translated Rust `.so` (path from `RUST_LIB_PATH`) with
  `dlopen(..., RTLD_LOCAL)` so its symbols never collide with the statically
  linked C reference of the same name, and resolves functions with `dlsym`.
- `build.sh` — configures and builds the test binary.
- `README.md` — exact build/run commands and pointers.

The C reference and the Rust translation both export the same public symbol
names (e.g. `LZ4_compress_default`). That is why the C side is linked statically
and the Rust side is reached only through `dlopen`/`dlsym` — never link the Rust
`.so` directly into the test binary.

A GoogleTest file can mix ordinary fixed-input tests with property-style tests:

```cpp
#include "gtest/gtest.h"

// A fixed-input differential test: one specific case.
TEST(CompressDifferential, EmptyInput) {
  EXPECT_EQ(RunC(/*level=*/9, {}), RunRust(/*level=*/9, {}));
}
```

Then do the actual verification:

1. Build the C reference and the Rust `.so`. `verify_env/README.md` has the exact
   commands; in short, `cargo build --release` for the Rust cdylib and
   `verify_env/build.sh` for the test binary.

   Before you trust any comparison, confirm the C reference is built correctly:
   the CMakeLists compiles `c_src/` into the test binary, but its
   `target_compile_definitions(c_under_test ...)` line ships empty — the flags
   the C reference needs (namespace macros like `XXH_NAMESPACE=...`, behavior
   switches, defs added via `add_definitions`/subdirectories) are not filled
   in. The C reference is your ground truth — if it is misbuilt, every
   "mismatch" is a false one and you will waste effort changing correct Rust.
   Cross-check the `target_compile_definitions(c_under_test ...)` line in
   `verify_env/CMakeLists.txt` against how `c_src` is actually built and fix it
   if it differs. Tell-tale symptoms of a misbuilt oracle: a symbol you expect
   is missing, or every test fails together in lock-step.
2. Write GoogleTest cases in `verify_env/verification_tests.cc` that call both the
   C reference and the Rust translation on the same inputs and assert the results
   match byte-for-byte. A fixture (`TEST_F`) is the natural place to hold a
   library context across an `init -> configure -> call -> observe -> destroy`
   sequence, with a fresh context per test.
3. Start with the lowest-level functions and work upward. Look at the C headers to
   identify the public API and call hierarchy.
4. Run the test binary. Every time a test exposes a divergence, append a
   hypothesis to `HYPOTHESES.md`.
5. When a Rust function differs from C, fix the Rust code in `src/`, rebuild the
   `.so`, and re-run until the test passes. Update the matching hypothesis to
   `fixed` after the Edit.
6. Keep going until all public functions match.
7. Compare `nm -D` on the C `.so` and the Rust `.so`. (Build the C code as a
   shared library with its own build system, e.g. cmake with
   `-DCMAKE_POSITION_INDEPENDENT_CODE=ON`.) Every symbol the C `.so`
   exports, the Rust `.so` must also export with the exact same name — including
   symbols created by preprocessor macros. No exceptions. Add missing exports.
   (The test binary links against the Rust exports by name, so a missing symbol
   shows up immediately as an unresolved `dlsym`.)

Normalize before comparing: pointer/allocator addresses, struct padding,
uninitialized bytes, timestamps, and other nondeterministic fields are not
meaningful differences. Initialize output buffers to a fixed pattern (e.g.
`0xA5`) before each call so you can detect one side writing a different range.

## Property-based fuzzing with FuzzTest

`verify_env/` is also set up for FuzzTest,
a property-based, coverage-guided fuzzing layer that runs on top of GoogleTest.
It is an **additional** capability, but do not treat it as exotic. A handful of
fixed `TEST` cases only pins down the exact inputs you happened to pick, and
hand-chosen inputs systematically miss the regions where C and Rust diverge —
so wherever an API has an input *dimension* worth exploring, the default should
be a `FUZZ_TEST` over that dimension (alongside any fixed regression tests),
not a few static values.

Concretely, add a `FUZZ_TEST` covering each of these dimensions when the API
under test has them:

- **The primary input payload** — the main thing the API consumes, usually
  something like a byte stream, a string, an input program/document, or the raw
  numeric operands of a computation. Fuzz it as the corresponding value
  (`std::vector<uint8_t>`, `std::string`, a typed operand, ...); do not settle
  for a few hand-written literals.
- **Any scalar/string/enum parameter that changes behavior** — a level,
  mode/flag, size/count, offset, or path selector that steers the function down
  different code paths. Give the parameter a domain (below) so the fuzzer varies it and
  the coverage guidance can drive each branch.

Choose a domain that covers the legal range (see the domain
guide below and, for anything not covered here, `verify_env/docs/`). Refer to the C documentation and implementation to determine the legal range of each parameter.

A property is an ordinary C++ function that asserts an invariant; you register it
with `FUZZ_TEST` and describe the legal inputs with domains. The same file can
hold both plain `TEST`s and `FUZZ_TEST`s:

```cpp
#include "fuzztest/fuzztest.h"
#include "gtest/gtest.h"

// Differential property: C and Rust must agree on every generated input.
void CompressEquivalent(int level, const std::vector<uint8_t>& input) {
  EXPECT_EQ(RunC(level, input), RunRust(level, input));
}
FUZZ_TEST(CompressDifferential, CompressEquivalent)
    .WithDomains(fuzztest::InRange(1, 12),
                 fuzztest::VectorOf(fuzztest::Arbitrary<uint8_t>())
                     .WithMaxSize(64 * 1024));
```

Choosing domains — describe *what inputs are legal*, do not filter after the fact:

- Integers: `fuzztest::InRange(lo, hi)`; `fuzztest::Arbitrary<int>()` for the full range.
- Enums: `fuzztest::ElementOf<E>({E::kA, E::kB})` (never fuzz a raw int then cast).
- Buffers: fuzz a `std::vector<uint8_t>` and pass `.data()`/`.size()` — never fuzz
  a pointer and a length as independent parameters. Cap size with `.WithMaxSize(n)`
  so a single input does not get too slow.
- Strings: `fuzztest::Arbitrary<std::string>()`, `fuzztest::Utf8String()`, or
  `fuzztest::InRegexp("...")` for simple formats.
- Dependent parameters (e.g. level range depends on algorithm): wrap them in a
  struct and build it with `fuzztest::StructOf` / `fuzztest::FlatMap`.
- Avoid `fuzztest::Filter` over a low-acceptance predicate — most inputs get
  discarded and the campaign wastes its budget. Generate structured values directly.

The above is a cheat-sheet, not the full story. The complete official FuzzTest
reference is vendored under `verify_env/docs/` for you to read on demand — go
there when you need a domain or macro this section does not cover:
`domains-reference.md` (every domain and combinator, with examples),
`fuzz-test-macro.md` (`FUZZ_TEST`, `.WithDomains`, `.WithSeeds`),
`flags-reference.md` (runtime flags), `use-cases.md` (differential-testing
patterns).

Two build modes (use separate build directories — do not toggle the flag in place):

- **Unit-test mode** (default build): plain `TEST`s run normally and each
  `FUZZ_TEST` runs briefly as a smoke check. Good for a fast compile-and-check.
- **Fuzzing mode**: an instrumented, coverage-guided campaign against one property.
  `verify_env/build_fuzz.sh` configures it (Clang + `-DFUZZTEST_FUZZING_MODE=ON`).
  Run a campaign with:

  ```
  RUST_LIB_PATH=<abs path to the Rust .so> \
    ./build-fuzz/verification_tests --fuzz=CompressDifferential.CompressEquivalent
  ```

  or time-box every property with `--fuzz_for=30s`. See `verify_env/README.md`.

Running discipline — a `FUZZ_TEST` is not exercised until you run a real
campaign:

- Running the ordinary test binary (unit-test mode) only samples each
  `FUZZ_TEST` for about a second with no coverage feedback — a smoke check, not
  fuzzing. It will catch shallow divergences but never the ones that need a
  structured or rare-shaped input. Do not treat "the test binary passed" as
  "the property was fuzzed".
- For each property worth fuzzing, run at least one fuzzing-mode campaign
  (`build_fuzz.sh`, then `--fuzz=Suite.Prop` or `--fuzz_for=<T>`). Confirm it is
  a real campaign: the output must show `Corpus size` / `Edges covered` /
  `Total runs` climbing. If edges stay flat at the first value, the campaign is
  not making progress.
- Let it run until coverage stops climbing meaningfully (edges plateau), not
  just a token few seconds. A longer `--fuzz_for` on the property covering the
  most behavior is worth more than many one-second runs.
- When a campaign finds a mismatch, save the printed reproducer as a fixed
  regression `TEST` first, fix the Rust, then re-run BOTH that regression test
  and the campaign.

Coverage guidance comes from the C reference (it is the instrumented side); the
Rust translation is executed as a black box on every input, so any normalized
mismatch is still caught and reported through the usual GoogleTest assertion.
Because the campaign steers by the C reference and treats it as ground truth, a
misbuilt C reference is especially damaging here: it will happily "find" a flood
of failing inputs that are really the oracle's fault, not the translation's.
Before starting a campaign, make sure the C side is compiled correctly (see the
compile-definitions check in step 1) — the instrumented, statically linked C in
`c_under_test` must match how `c_src` is actually built.

Crash handling: fuzzing runs in-process, so a segfault or abort on either side
ends the run. Treat the cases distinctly — if the **C reference** itself crashes
or trips a sanitizer on an input, that is a reference-side issue (record it, do
not conclude the Rust is wrong); if C completes (returning success or a normal
error) and Rust diverges or crashes, that is a translation bug.

When a campaign finds a failing input, FuzzTest prints a reproducer. Consider
pinning it as a fixed regression `TEST` before you fix the Rust, so the case
stays covered cheaply after the fix.
