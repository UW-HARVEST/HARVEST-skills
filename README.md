# HARVEST Skills: Agentic C-to-Rust Translation

Agent Skills for translating C codebases into pure Rust with byte-identical
behavior, and for verifying the result against the original C as ground truth.

These skills distill the methodology of
[UW-HARVEST/ACTOR](https://github.com/UW-HARVEST/ACTOR) ("Agentic C-to-Rust
translation", part of the DARPA TRACTOR program research at UW) into two
plain-markdown skills that work with any coding agent supporting the
[Agent Skills](https://agentskills.io) standard. They have been
exercised on real-world libraries such as zstd, lz4, and
SPHINCS+ reference code.
The largest translation to date is the 80kLoC zstd.

## The two skills

### `c-to-rust-translation`

Translates a C project into a pure-Rust Cargo package. The core problem it
addresses: **translation is sensitive to context-window compaction**. Unlike bug fixing, it needs line-level recall of source that easily fills up the context window. The skill therefore enforces:

- Self-assessment and cheap reconnaissance first
- A persistent `PLAN.md`
- Aggressive sub-agent delegation

### `c-to-rust-verification`

Verifies and fixes a finished translation by differential testing:

- Builds the original C as a shared library and compares it against the Rust
  via `libloading`-based integration tests, function by function,
  byte-for-byte, plus `nm -D` symbol-parity checks and whole-binary stdout
  comparison.
- Maintains a persistent `HYPOTHESES.md` bug-hypothesis log
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

## License

[MIT](LICENSE) © HARVEST Developers
