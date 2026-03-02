# Show HN: Don't smoke tokens, grep content — self-documenting monorepo for AI agents

I have a Go monorepo — 22 services, 590 `.go` files, 97K lines. Every dev session used to start with a 2-hour briefing: 4 screens, 3 Claude instances + 1 Gemini doing exploratory reads, burning ~1M tokens just to produce a dev plan. The plan gets compacted, then "implement this."

**The fix**: two "skill files" — structured prompts that forbid coding and force systematic documentation. No tooling, no build step — just Go comments and ASCII art scannable by `grep`.

## Skill 1 — Codebase audit (annotations + manifests)

One session produced:

- **540** `CLAUDE:SUMMARY` annotations (one per `.go` file — scannable by `grep`, replaces reading the file)
- **94** `CLAUDE:WARN` annotations (non-obvious traps: locks, goroutines, silent errors)
- **12** CLAUDE.md manifests refactored (40-60 lines, strict format)
- **542** INDEX.map entries (auto-generated lookup table)

The annotations are plain Go comments. `grep -rn "CLAUDE:SUMMARY" siftrag/` gives you an entire service in 30 seconds.

## Skill 2 — Schematics (ASCII architecture diagrams)

A second skill generates `*_schem.md` files — ASCII art technical schemas for every service and package. One session (112K tokens, 7 minutes) rewrote the ecosystem schema (300 lines) and corrected 4 local schemas.

Each schema documents architecture, data flow, SQL DDL, and API surface — visually, without opening source code. Example: a 14-file router package with 260+ lines in `router.go` alone gets a [214-line ASCII schema](example-schem.md) covering the dispatch logic, hot-reload loop, transport factories, circuit breaker state machine, and middleware chain. An agent reads this instead of 14 files.

## The final state: minimal CLAUDE.md + grepped content

After both skills, an agent working on any service sees 3 layers:

1. **CLAUDE.md** (~50 lines) — responsibility, deps, invariants, traps. The manifest.
2. **`*_schem.md`** (~200 lines) — ASCII architecture, SQL schema, data flow. The blueprint.
3. **`CLAUDE:SUMMARY` + `CLAUDE:WARN`** in source — grepped, never read in full. The index.

The agent's workflow becomes: `cat CLAUDE.md` → `grep SUMMARY` → `grep WARN` → read 20 targeted lines. No browsing, no `find`, no "let me explore the codebase."

## The chaining problem (and fix)

Claude Code injects the root CLAUDE.md into the main conversation, but sub-agents start blank. An agent receiving "plan X in siftrag" reads `siftrag/CLAUDE.md` but never goes up to root. It misses the research protocol and the architecture schemas.

Fix: each local CLAUDE.md starts with 3 lines — the mandatory `grep` commands + an explicit **ban** on browsing tools. Without the ban line, agents acknowledge the protocol but still fall back to `find *.go` + Read every file. With it, they grep.

```markdown
> **Protocol** — Before any task, read [`../CLAUDE.md`](../CLAUDE.md) §Research protocol.
> Required commands: `cat <dir>/CLAUDE.md` → `grep -rn "CLAUDE:SUMMARY"` → `grep -n "CLAUDE:WARN" <file>`.
> **Forbidden**: Glob/Read/Explore/find instead of `grep -rn`. Never read an entire file as first action.
```

## A/B test

Same prompt ("audit sherpapi integration in siftrag"), fresh terminal:

- **With full doc system**: **2 minutes, 58K tokens**. Zero sub-agents. Follows grep protocol. Correctly identifies the dormant middleware as *intentional design*.
- **Without root CLAUDE.md**: **8 minutes, 73K tokens**. Launches 2 Explore sub-agents, `find *.go` + Read every file. Reports 6 "bugs" including a P1 that's actually the intended dormant pattern. **Misclassifies design intent as a bug.**

The root CLAUDE.md isn't just navigation — it's architectural context that prevents false positives.

**Repo**: https://github.com/hazyhaar/GenAI_paterns — skill templates, example report, example schema, annotation format spec. MIT.
