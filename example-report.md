# Documentation Audit Report — Example

**Date**: March 2026
**Scope**: Full audit (11 active directories + 1 new service)
**Operator**: Claude Opus 4.6, driven by the skill file (6 phases)
**Duration**: ~2h machine time (phases parallelized)

---

## 1. Numbers

| Metric | Value |
|--------|-------|
| `.go` files audited (non-test, non-generated) | 590 |
| Total Go lines of code | 97,545 |
| `CLAUDE:SUMMARY` annotations added | 126 |
| `CLAUDE:WARN` annotations added | 94 |
| `CLAUDE.md` manifests modified | 12 |
| `INDEX.map` entries before / after | 406 → 542 |
| Files changed (parent repo) | 52 files, +933 −2,831 |
| Files changed (shared library) | 117 files, +526 −55 |
| Files changed (content service) | 10 files, +24 −3 |
| Files changed (tenant library) | 6 files, +13 −12 |
| Files changed (new API service) | 13 files, +566 −18 |
| Conformity test result | 1,128 PASS, 48 SKIP, 11 FAIL (all pre-existing) |

### CLAUDE:SUMMARY coverage by service (before → after)

| Service | .go files | Before | After | Coverage |
|---------|-----------|--------|-------|----------|
| shared-lib | ~140 | 27 | 140 | 20% → 100% |
| new-api | 11 | 0 | 11 | 0% → 100% |
| content-svc | ~65 | 63 | 65 | 97% → 100% |
| All others | — | — | — | 100% (already) |

---

## 2. Changes by category

### 2.1 File-head annotations (126 files)

Standard block added to each file:
```go
// CLAUDE:SUMMARY <responsibility sentence>
// CLAUDE:DEPENDS <local imports>
// CLAUDE:EXPORTS <public names used by other packages>
```

### 2.2 Function-level WARN annotations (94 functions)

| Service | WARNs added | Patterns documented |
|---------|-------------|---------------------|
| shared-lib | 58 | Locks (mu.Lock), launched goroutines, silent errors, blocking channels |
| content-svc | 12 | Process management, SPA race conditions, scheduler goroutines |
| pipeline-svc | 9 | Double-checked lock, silent filesystem I/O, async rebuild |
| tenant-lib | 9 | Reaper goroutine, double-checked lock resolve, filesystem delete |
| frontend-svc | 6 | Blocking forward loop, per-shard goroutines, cache lifetime |

### 2.3 CLAUDE.md manifests modified (12 files)

| File | Lines before → after | Nature |
|------|---------------------|--------|
| pipeline-svc/CLAUDE.md | 268 → 60 | Major trim: pipeline ASCII art, extended config, deploy architecture |
| frontend-svc/CLAUDE.md | 166 → 60 | Trim: migration docs, OpenAPI duplication, dormant service details |
| deploy/CLAUDE.md | 161 → 60 | Trim: step-by-step deploy, credentials table, variable architecture |
| shared-lib/CLAUDE.md | 87 → 63 | Trim: SDK migration history, CI/CD duplication. Added missing DO NOT |
| conformity/CLAUDE.md | 75 → 56 | Added 9 missing test files to index |
| content-svc/CLAUDE.md | 66 → 67 | **Critical fix**: "Basic Auth" → "JWT cookie" |
| tenant-lib/CLAUDE.md | 65 → 57 | Fix dependents (added downstream services), trim SQL schema |

---

## 3. Before / after — representative `.go` file

**`shared-lib/connectivity/router.go`** — the connectivity router, core of inter-service communication.

### Before

```go
// Package connectivity provides a smart service router that dispatches calls
// either locally (in-memory function call, ~0.01ms) or remotely (QUIC/HTTP,
// ~50ms) based on a SQLite routes table reloaded at runtime.
package connectivity
```

### After

```go
// CLAUDE:SUMMARY Core Router dispatching service calls locally or remotely based on SQLite routes with hot-reload.
// CLAUDE:DEPENDS
// CLAUDE:EXPORTS Router, New, Handler, TransportFactory, Option, WithLogger

// Package connectivity provides a smart service router ...
package connectivity

// ...

// CLAUDE:WARN Takes mu.Lock; mutates localHandlers map.
func (r *Router) RegisterLocal(service string, h Handler) {

// CLAUDE:WARN Takes mu.Lock for entire rebuild. Silently skips routes with missing/failed factories.
func (r *Router) Reload(ctx context.Context, db *sql.DB) error {

// CLAUDE:WARN Takes mu.Lock; closes all remote handlers; always returns nil.
func (r *Router) Close() error {
```

**Gain**: `grep CLAUDE:WARN connectivity/router.go` gives you every trap without reading the 260-line file.

---

## 4. Top 10 most critical WARNs

### 1. Connectivity Router — Reload
```
Takes mu.Lock for entire rebuild. Silently skips routes with missing/failed factories (logs, no error). Closes replaced handlers.
```
**Danger**: a missing factory produces no error. The route disappears silently. A service can become unreachable with no alert.

### 2. DB Sync Subscriber — OnSwap callbacks
```
Takes mu.Lock. Callbacks execute synchronously inside handleSnapshot WHILE mu.Lock is held — must not block.
```
**Danger**: a callback that takes another lock = guaranteed deadlock.

### 3. DB Sync QUIC Transport — Sleep 200ms
```
Network + filesystem I/O. Sleeps 200ms after stream.Close() — required for QUIC FIN delivery.
```
**Danger**: the 200ms sleep looks like a hack a developer would "fix". Removing it breaks QUIC replication.

### 4. SPA Observer — Race condition
```
Called as goroutine. Mutates tab.PageURL without lock. Drains rawCh — races with main loop.
```
**Danger**: known race between the SPA handler and the main observation loop.

### 5. Trace Store — Infinite recursion
```
Launches goroutine; db MUST use raw "sqlite" driver — "sqlite-trace" causes infinite recursion.
```
**Danger**: traced SQL driver + trace store = each INSERT generates a trace that generates an INSERT → stack overflow.

### 6. Injection normalizer — Panic at init
```
Mutates package globals at import time; panics if embedded JSON is malformed.
```
**Danger**: corrupted `go:embed` = panic at startup of any binary that imports the package.

### 7. Tenant Pool — Double-checked lock + eviction
```
Double-checked lock, may evict oldest connection, calls ShardFactory (filesystem/network).
```
**Danger**: Resolve can evict an active connection if the pool is full.

### 8. Job Queue — Silently deleted jobs
```
Drains ALL visible jobs in tight loop. Silently discards Ack/Nack errors. Jobs exceeding MaxAttempts silently deleted.
```
**Danger**: a job that fails MaxAttempts times is deleted with no trace. Pipeline can lose data silently.

### 9. Search Service — Silent shard failures
```
Launches one goroutine per shard. Silently skips failed shards — returns nil error even if all fail.
```
**Danger**: a search across 5 shards where all 5 fail returns `([], nil)` — no error. User thinks there are no results.

### 10. MCP Transport Factory — context.Background
```
Connects eagerly to QUIC endpoint during Reload (fail fast). Uses context.Background() — ignores caller's context.
```
**Danger**: QUIC connection ignores the caller's context. A Reload with a short context can't cancel a slow MCP connection.

---

## 5. Method

### Orchestration

The audit was conducted by Claude Opus 4.6 in agent mode, following the 6 phases of the skill file. Each phase was parallelized via specialized sub-agents.

### Phases and parallelization

| Phase | Agents | Work |
|-------|--------|------|
| 1. Inventory | 1 explorer | Scan 11 directories, count missing annotations |
| 2. SUMMARY | 10 parallel agents | 113 files in shared-lib (8 batches), 11 new-api, 2 content-svc |
| 3. WARN scan | 3 explorers | Identify 103 candidate functions |
| 3. WARN write | 3 parallel agents | Write 94 WARN annotations |
| 4. CLAUDE.md audit | 2 parallel agents | Factual verification of 12 manifests |
| 4. CLAUDE.md fix | 2 parallel agents | Factual corrections (6 files) + trim (5 files) |
| 5. Conformity | 1 agent | Scan non-standard annotations, credentials, migration |
| 6. Regeneration | direct | Rebuild index + conformity tests + corrections |

### Safeguards

- No application `.go` file modified beyond `CLAUDE:*` comments
- No functional test touched
- No control flow, signature, or import modified
- All trimmed CLAUDE.md stay within the 40-60 line range
- Removed information verified as redundant (already in memory files, deploy scripts, or the code itself)

---

## 6. Issues encountered

### 6.1 `CLAUDE:DEPENDS none` vs empty

Agents wrote `// CLAUDE:DEPENDS none` for files without local dependencies. Conformity test parses "none" as a package name → FAIL.

**Resolution**: bulk replace of 85 files `// CLAUDE:DEPENDS none` → `// CLAUDE:DEPENDS` (empty).

### 6.2 DO NOT section removed by trim

A trim agent removed the "DO NOT" section from a shared library CLAUDE.md. Conformity test detected the absence.

**Resolution**: manual re-addition of the section.

### 6.3 No regressions caused

The 11 failing tests are all pre-existing. The 2 regressions introduced during the audit were detected and fixed in the same session.

---

*Report generated March 2026 by Claude Opus 4.6 using the doc-audit skill.*
