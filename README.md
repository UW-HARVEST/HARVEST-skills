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

- Ships a ready-to-use GoogleTest environment. It compiles the original C into
  the test binary as the oracle and loads the translated Rust as a shared
  library. Each test runs both sides and compares the results byte for byte.
- Adds coverage-guided [FuzzTest](https://github.com/google/fuzztest)
  properties over each input dimension.
- Also checks symbol parity with `nm -D`.
- Keeps a persistent hypothesis log (`HYPOTHESES.md`) with one entry per bug
  hypothesis: open, confirmed, refuted, or fixed. The log survives context
  compaction, so the agent does not investigate the same bug twice.
- Delegates code-heavy fixes to sub-agents. The main agent runs the tests,
  compares the output, and updates the hypothesis log.

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

## How to use

To use a skill, you typically describe to your agent the task in plain words, for example
"read the C-to-Rust translation skill and use it on this project". Most agents
find the skill and start on their own. Some agents also accept an explicit
call, such as `/c-to-rust-translation` in Claude Code. For that syntax, see the
documentation of your own agent.

We recommend running the two skills in sequence. Each skill also works alone.

1. Put the C project in its own directory. Tell the agent where it is and where to put the Rust output.
2. Run `c-to-rust-translation`.
3. Run `c-to-rust-verification`.

If the result is not good enough, try running the verifier again. Add a hint about where to look, for
example, give an input that produces different output.

## License

[MIT License](LICENSE) © HARVEST Developers
