# HARVEST Skills — Agentic C-to-Rust Translation

Agent Skills for translating C codebases into pure Rust with byte-identical
behavior, and for verifying the result against the original C as ground truth.

These skills distill the methodology of
[UW-HARVEST/ACTOR](https://github.com/UW-HARVEST/ACTOR) ("Agentic C-to-Rust
translation", part of the DARPA TRACTOR program research at UW) into two
plain-markdown skills that work with any coding agent supporting the
[Agent Skills](https://agentskills.io) standard — Claude Code, OpenAI Codex,
Cursor, GitHub Copilot, Gemini CLI, OpenCode, and others. They have been
exercised on real-world libraries such as lz4, zstd, jansson, and
SPHINCS+ reference code.

## The two skills

### `c-to-rust-translation`

Translates a C project into a pure-Rust Cargo package. The core problem it
addresses: **translation is uniquely sensitive to context-window compaction**
— unlike bug fixing, it needs line-level recall of source that may be far
larger than the context window. The skill therefore enforces:

- **Self-assessment and cheap reconnaissance first** — size the codebase with
  `wc`/`grep` before reading anything, and pick a strategy that fits the
  window.
- **A persistent `PLAN.md`** with a byte-stable Invariants section, checkboxed
  subtasks sized to fit one uncompacted window, and a recovery protocol: after
  any compaction, `cat PLAN.md` comes first.
- **Aggressive sub-agent delegation** — the main agent owns the plan and the
  build; source reading and module writing go to sub-agents so large code
  never lives in the coordinator's context.
- **Fidelity invariants** — exact C ABI symbols after macro renaming,
  bug-for-bug behavioral fidelity, build-time configurability preserved as
  Cargo features, and a hard ban on `cc`-crate wrapping or `*-sys` crate
  shortcuts.

### `c-to-rust-verification`

Verifies and fixes a finished translation by differential testing:

- Builds the original C as a shared library and compares it against the Rust
  via `libloading`-based integration tests, function by function,
  byte-for-byte — plus `nm -D` symbol-parity checks and whole-binary stdout
  comparison.
- Maintains a persistent **`HYPOTHESES.md`** bug-hypothesis log
  (open/confirmed/refuted/fixed) so investigation state survives compaction
  and the agent never rediscovers the same bug twice.
- Delegates code-heavy fixes to sub-agents while the main agent owns
  execution, comparison, and the hypothesis log.

They are designed to run in sequence (translate, then verify) but each also
works on its own.

## Installation

**Claude Code (as a plugin):**

```
/plugin marketplace add UW-HARVEST/HARVEST-skills
/plugin install harvest-skills@harvest-skills
```

**Any agent, via [skills.sh](https://skills.sh):**

```bash
npx skills add UW-HARVEST/HARVEST-skills
```

**Manual (any Agent Skills–compatible tool):** copy the directories under
`skills/` into your skills location, e.g. `~/.claude/skills/`,
`.claude/skills/`, or the cross-agent `~/.agents/skills/`.

## Usage

The skills trigger automatically when you ask a coding agent to translate or
port C code to Rust (or to verify such a translation), or invoke them
explicitly:

```
/c-to-rust-translation   Translate the C library in ./c_src into Rust.
/c-to-rust-verification  Verify the Rust translation in this directory against ./c_src.
```

Conventions the skills assume (adaptable per project): the C source sits in
its own directory (examples use `c_src/`) and is never modified; the Rust
crate is created in the working directory.

## Layout

```
skills/
├── c-to-rust-translation/
│   ├── SKILL.md
│   └── references/plan-template.md        # PLAN.md template incl. Invariants
└── c-to-rust-verification/
    ├── SKILL.md
    └── references/hypotheses-template.md  # HYPOTHESES.md template incl. Invariants
```

## License

[MIT](LICENSE) © HARVEST Developers
