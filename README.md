# claude-doc-audit

A structured prompt ("skill file") that turns a codebase into a self-documenting system for AI agents. No tooling, no build step — just Go comments and markdown manifests scannable by `grep`.

## The problem

Large codebases are opaque to AI agents. Every session starts with expensive exploratory reads — the agent opens files, scans directories, builds a mental model from scratch. In a 590-file Go monorepo, this costs ~1M tokens and 2 hours before any real work begins.

Worse: agent sub-processes (like "plan this feature") don't inherit the parent's context. They start blank, read one local file, and dive into source code. They never discover the architecture docs, the research protocol, or the annotation index sitting one directory up.

## The solution

A **skill file** — a structured prompt in 6 phases that forbids coding and forces a systematic documentation audit. Run it once (or periodically), and your codebase becomes self-documenting:

### What it produces

| Artifact | Purpose |
|----------|---------|
| `CLAUDE:SUMMARY` | One-line file description. One per `.go` file, line 1. `grep -rn "CLAUDE:SUMMARY" service/` gives you the entire service in 30 seconds. |
| `CLAUDE:WARN` | Non-obvious trap above a function. Locks, goroutines, silent errors, init-order dependencies. What the code *doesn't* tell you. |
| `CLAUDE:DEPENDS` | Local imports (not stdlib). Scannable dependency graph without reading imports. |
| `CLAUDE:EXPORTS` | Public names actually used by other packages. Not everything exported — what's imported. |
| `CLAUDE.md` | Per-directory manifest: responsibility, dependencies, build, invariants, traps. 40-60 lines max. |
| `INDEX.map` | Auto-generated global lookup table from all `CLAUDE:SUMMARY` annotations. |

### What it looks like

**Before** — a 260-line router file:
```go
// Package connectivity provides a smart service router...
package connectivity
```

**After**:
```go
// CLAUDE:SUMMARY Core Router dispatching service calls locally or remotely based on SQLite routes with hot-reload.
// CLAUDE:DEPENDS
// CLAUDE:EXPORTS Router, New, Handler, TransportFactory, Option, WithLogger

// Package connectivity provides a smart service router...
package connectivity

// CLAUDE:WARN Takes mu.Lock for entire rebuild. Silently skips routes with missing/failed factories.
func (r *Router) Reload(ctx context.Context, db *sql.DB) error {

// CLAUDE:WARN Takes mu.Lock; closes all remote handlers; always returns nil.
func (r *Router) Close() error {
```

`grep CLAUDE:WARN connectivity/router.go` gives you every trap without reading the file.

## The chaining problem

Claude Code (the CLI) injects your root `CLAUDE.md` into the main conversation. But sub-agents — launched via the Agent tool for parallel work — start with a blank context. They only see their prompt and whatever they choose to read.

An agent receiving "plan rate limiting in siftrag" reads `siftrag/CLAUDE.md` and goes straight to source code. It never discovers:
- The research protocol (5 mandatory `grep` commands before reading any file)
- `INDEX.map` (542-entry lookup table)
- Architecture schemas (`*_schem.md`)
- Cross-service annotations (`CLAUDE:WARN` in shared libraries)

### The fix: protocol chaining

Each local `CLAUDE.md` starts with 2 lines:

```markdown
> **Protocol** — Before any task, read [`../CLAUDE.md`](../CLAUDE.md) §Research protocol.
> Required commands: `cat <dir>/CLAUDE.md` → `grep -rn "CLAUDE:SUMMARY"` → `grep -n "CLAUDE:WARN" <file>`.
```

Three options were considered:
1. **One-line link** — too passive, agents don't follow links reliably
2. **Inline all 17 lines** — guaranteed but 204 duplicated lines across 12 files
3. **3 key commands + link** — inlines the essential `grep` commands, links to root for the rest

Option (c) was chosen. The 3 inlined commands are enough for the agent to start the protocol; the link provides the full context if needed.

### A/B test results

Same prompt, same task, fresh terminal session:

| Metric | Before chaining | After chaining |
|--------|----------------|----------------|
| First action | `Read` / `Glob` (browsing) | `cat CLAUDE.md` (protocol) |
| Uses `grep CLAUDE:SUMMARY` | No | Yes |
| Discovers root protocol | No | Yes |
| Reads files before grepping | Yes (entire files) | No (grep first, targeted read) |

## The skill file

See [`skill-codebase-audit.md`](skill-codebase-audit.md) for the complete template. It's designed for Go monorepos but the structure is language-agnostic.

### 6 phases

1. **Inventory** — scan directories, count missing annotations
2. **SUMMARY annotations** — one per file, responsibility in one sentence
3. **WARN annotations** — above functions with non-obvious traps
4. **CLAUDE.md audit** — verify manifests against reality, trim to 40-60 lines
5. **Conformity** — migrate stale annotations, check for credentials
6. **Regeneration** — rebuild index, run conformity tests

### What it forbids

The skill explicitly forbids coding, bug fixing, refactoring, or touching tests. It's a documentation-only session. This constraint is essential — without it, the agent will "fix" things it finds during the audit.

## Sample numbers

From a real audit of a 590-file, 97K-line Go monorepo:

| Metric | Value |
|--------|-------|
| `CLAUDE:SUMMARY` annotations | 540 |
| `CLAUDE:WARN` annotations | 94 |
| `CLAUDE.md` manifests refactored | 12 |
| `INDEX.map` entries | 542 |
| Files changed | 198 |
| Lines added/removed | +2,062 / −2,919 |
| Conformity tests | 1,128 PASS, 48 SKIP, 11 FAIL (pre-existing) |

### A/B test: same prompt, same codebase, fresh terminal

Prompt: "audit sherpapi integration in siftrag — is it correctly implemented?"

| Metric | With full doc system | Without root CLAUDE.md |
|--------|---------------------|----------------------|
| Time | **2 minutes** | **8 minutes** |
| Tokens | **58K** (29% of context) | **73K** (36% of context) |
| Sub-agents launched | 0 | 2 (brute-force Explore) |
| Permission prompts | 0 | Dozens |
| First action | `cat CLAUDE.md` → `grep CLAUDE:SUMMARY` | `find *.go` → Read every file |
| Verdict | "Integration correct, dormant mode functional" | "6 bugs found" (including P1) |
| False positives | **0** | **≥1** (dormant pattern flagged as P1 bug) |

The root CLAUDE.md isn't just navigation — it provides the architectural context that prevents false positives. Without it, agents misclassify design intent as bugs.

**~17x reduction in tokens, ~60x in wall time.**

## Files in this repo

| File | Description |
|------|-------------|
| [`skill-codebase-audit.md`](skill-codebase-audit.md) | The skill file template (generic, ready to use) |
| [`example-report.md`](example-report.md) | Anonymized audit report from a real run |
| [`annotation-format.md`](annotation-format.md) | Specification of the annotation format |
| [`SHOW_HN.md`](SHOW_HN.md) | The Hacker News post |

## License

MIT — see [LICENSE](LICENSE).
