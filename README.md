# HARVEST Skills: Agentic C-to-Rust Translation

Agent Skills for translating C codebases into pure Rust with byte-identical
behavior, and for verifying the result against the original C as ground truth.

These skills distill the methodology of the [ACTOR](https://homes.cs.washington.edu/~mernst/pubs/c-to-rust-ieeesp2026-abstract.html) state-of-the-art C-to-Rust translator.
[UW-HARVEST/ACTOR](https://github.com/UW-HARVEST/ACTOR) is the active research version.
This repository is the frozen distribution version.
It drops the prompts that adapt to specific test benches
and in-development features.
It provides two skills that work with any coding agent supporting the
[Agent Skills](https://agentskills.io) standard.

## The two skills

### `c-to-rust-translation`

Translates a C project into a pure-Rust Cargo package. The skill addresses one
core problem: translation is sensitive to context compaction. Unlike bug
fixing, translation needs line-level recall of the C source. That source fills
the context window fast. The skill therefore:

- Checks its own context window and output limit first.
- Estimates the size of the C project without reading the files.
- Keeps a persistent `PLAN.md` that survives context compaction.
- Delegates file-level translation to sub-agents.

### `c-to-rust-verification`

Verifies and fixes a finished translation by differential testing:

- Builds the original C as a shared library. It then compares the C library and
  the Rust package function by function, byte for byte, with
  `libloading`-based integration tests.
- Also checks symbol parity with `nm -D`, and compares whole-binary stdout.
- Keeps a persistent hypothesis log (`HYPOTHESES.md`) with one entry per bug
  hypothesis: open, confirmed, refuted, or fixed. The log survives context
  compaction, so the agent does not investigate the same bug twice.
- Delegates code-heavy fixes to sub-agents. The main agent runs the tests,
  compares the output, and updates the hypothesis log.

Run the two skills in sequence: translate, then verify. Each skill also works
alone.

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

**Manual (any tool that supports Agent Skills):** copy the directories under
`skills/` into your skills directory, for example `~/.claude/skills/`,
`.claude/skills/`, or the cross-agent `~/.agents/skills/`.

## License

[MIT License](LICENSE) © HARVEST Developers
