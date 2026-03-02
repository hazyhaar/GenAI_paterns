# Show HN: I made my 2.7 GB monorepo self-documenting for AI agents

I have a Go monorepo — 22 services, 590 `.go` files, 97K lines. Every dev session used to start with a 2-hour briefing: 4 screens, 3 Claude instances + 1 Gemini doing exploratory reads, burning ~1M tokens just to produce a dev plan. The plan gets compacted, then "implement this."

**The fix**: a "skill file" — a structured prompt in 6 phases that forbids coding and forces a systematic documentation audit. One session produced:

- **540** `CLAUDE:SUMMARY` annotations (one per `.go` file — scannable by `grep`, replaces reading the file)
- **94** `CLAUDE:WARN` annotations (non-obvious traps: locks, goroutines, silent errors)
- **12** CLAUDE.md manifests refactored (40-60 lines, strict format)
- **542** INDEX.map entries (auto-generated, global lookup table)

The annotations are plain Go comments. No tooling dependency, no build step. `grep -rn "CLAUDE:SUMMARY" siftrag/` gives you the entire service in 30 seconds.

**The hidden problem — agents don't inherit context.** Claude Code injects the root CLAUDE.md into the main conversation, but sub-agents (launched via the Agent tool) start blank. They only see their prompt + whatever they choose to read. An agent receiving "plan X in siftrag" reads `siftrag/CLAUDE.md` but never goes up to root. It misses the research protocol, INDEX.map, the architecture schemas. It dives into source like the documentation doesn't exist.

**The fix for the fix**: each local CLAUDE.md now starts with 3 lines — the mandatory `grep` commands, a link to the full protocol, and critically, an explicit **ban** on the tools agents default to (`Glob`, `Read`, `Explore`, `find`). Without the ban line, agents acknowledge the protocol but still fall back to browsing. With it, they use `grep` as intended.

```markdown
> **Protocol** — Before any task, read [`../CLAUDE.md`](../CLAUDE.md) §Research protocol.
> Required commands: `cat <dir>/CLAUDE.md` → `grep -rn "CLAUDE:SUMMARY"` → `grep -n "CLAUDE:WARN" <file>`.
> **Forbidden**: Glob/Read/Explore/find instead of `grep -rn`. Never read an entire file as first action.
```

**A/B test**: same prompt ("audit sherpapi integration in siftrag"), same codebase, fresh terminal. Two runs:

- **With full doc system** (annotations + root CLAUDE.md + chaining): **2 minutes, 58K tokens**. Zero sub-agents launched. The agent follows the grep protocol, reads targeted fragments, produces a 10-point audit with 4 non-blocking observations. Correctly identifies the dormant middleware as *intentional design*, not a bug.

- **Without root CLAUDE.md** (annotations present, chaining present, but root protocol removed): **8 minutes, 73K tokens**. Launches 2 Explore sub-agents that `find *.go` + Read every file individually. Dozens of permission prompts. Reports 6 "bugs" — including a P1 "phantom activation" that is actually the intended dormant pattern. **The agent misclassifies design intent as a bug** because it lacks the architectural context the root CLAUDE.md provides.

The root CLAUDE.md isn't just a navigation protocol — it's the architectural context that prevents false positives. Without it, agents over-flag design decisions they don't understand.

**What's in the repo**: the skill file (generic template), a sample audit report, the annotation format spec. MIT licensed.

Repo: https://github.com/hazyhaar/claude-doc-audit
