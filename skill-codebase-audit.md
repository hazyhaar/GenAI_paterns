---
name: doc-audit
description: Audit and maintain codebase documentation. Run in a dedicated session, never during development. Invoked when you need a documentation refresh, a codebase audit, or annotation sync. Produces annotations, updates CLAUDE.md manifests, never touches application code.
---

Dedicated session. You don't code, you don't fix, you don't refactor. You read and you document.

## Annotations

Four annotations. No more. Every `.go` file (non-generated, non-test) starts with a contiguous block:

```go
// CLAUDE:SUMMARY One sentence. Single responsibility of the file.
// CLAUDE:DEPENDS pkg/path1, pkg/path2
// CLAUDE:EXPORTS PublicName1, PublicName2
```

`CLAUDE:WARN` is different. It goes above each dangerous function, not at file head:

```go
// CLAUDE:WARN takes mu.Lock — blocks all Search. Mutates: flatVecs, cache, centroid, nextID
func (idx *Index) Insert(vecs [][]float32, ids [][]byte) error {
```

A WARN documents what the code doesn't say obviously:
- Locks taken and their scope (what other functions are blocked)
- State mutated beyond the function's return values
- Silent errors (return without log, without error returned)
- Startup order or runtime dependencies
- Deployment constraints (shared ports, restart sequences)
- Counter-intuitive decisions (why 2*R and not R)

A WARN is NOT: a TODO, an optimization idea, a godoc comment, a rephrasing of what the code already says.

## Sequence

### Phase 1 — Inventory

```bash
# List active directories
grep -n "│" CLAUDE.md | head -30

# For each active directory:
grep -rn "CLAUDE:SUMMARY" --include="*.go" <dir>/
```

Identify gaps: `.go` files without SUMMARY.

```bash
# Go files (non-test, non-generated) without annotation
find <dir> -name "*.go" ! -name "*_test.go" ! -name "*.pb.go" ! -name "*_templ.go" | while read f; do
  grep -q "CLAUDE:SUMMARY" "$f" || echo "MISSING: $f"
done
```

### Phase 2 — File-head annotations

For each file without SUMMARY or with stale SUMMARY:

1. `grep -n "^func " <file>` — list public functions
2. `head -10 <file>` — see imports and package
3. Write the SUMMARY + DEPENDS + EXPORTS block

Rules:
- SUMMARY = one sentence, not two. Starts with a verb or noun.
- DEPENDS = local imports (not stdlib, not third-party). Path relative to module.
- EXPORTS = public names actually used by other packages. Not everything exported — what's imported. When in doubt: `grep -rn "PublicName" --include="*.go" ../ | grep -v <current_dir>`.

### Phase 3 — WARN per function

For each file, `grep -n "^func " <file>` then read each function.

Add a WARN above if the function:
- Takes a lock (mu.Lock, mu.RLock, rebuildMu)
- Mutates state beyond its parameters (struct fields, globals, cache)
- Does a silent return on error (no log, no error returned)
- Launches a goroutine
- Has a constrained call order (must be called after X)
- Touches network, filesystem, or a shared port
- Contains a counter-intuitive decision that would be "fixed" by a naive developer

Do NOT add a WARN if the function is a pure getter, a side-effect-free helper, or if the behavior is obvious from the signature.

### Phase 4 — Local CLAUDE.md

Check each active directory's `CLAUDE.md`:
- Are the listed invariants still true? (`grep -rn` to verify)
- Is there a trap discovered during phases 2-3 that's missing?
- Is the build/test still correct? (`make test` if Makefile present)
- Do the listed dependencies match the `CLAUDE:DEPENDS`?

A local CLAUDE.md is 40-60 lines max. If it's longer, something drifted.

### Phase 5 — Conformity and migration

The documentation system has a defined format. Content outside the format isn't necessarily useless — it's often a previous Claude session that documented something in the wrong place or format.

**Principle: sort, migrate, then clean. Never delete without migrating useful information.**

**In `.go` files** — only these `CLAUDE:` annotations are legitimate:
- `// CLAUDE:SUMMARY` — one per file, line 1, one sentence
- `// CLAUDE:DEPENDS` — one per file, after SUMMARY
- `// CLAUDE:EXPORTS` — one per file, after DEPENDS
- `// CLAUDE:WARN` — above a function, never at file head

```bash
# Find any non-standard CLAUDE: annotation
grep -rn "CLAUDE:" --include="*.go" <dir>/ | grep -v "CLAUDE:SUMMARY\|CLAUDE:DEPENDS\|CLAUDE:EXPORTS\|CLAUDE:WARN"
```

For each result:
1. Read the content — is it useful information in the wrong format?
2. If yes: migrate to the right annotation (WARN if it's a trap, SUMMARY if it's a description) or to the local CLAUDE.md (if it's a package invariant)
3. Only after migration: delete the non-standard annotation

**In CLAUDE.md files** — manifest format:
- Responsibility (one sentence)
- Dependencies and dependents
- Key files / key types
- Build / test / deploy
- Invariants and known traps

For each out-of-section content:
1. Is it a trap or invariant? → migrate to "Invariants and known traps"
2. Is it build/deploy info? → migrate to "Build / test / deploy"
3. Is it a discovery about a specific function? → migrate to `CLAUDE:WARN` on the function in the `.go` file
4. Is it noise without information? → delete
5. If in doubt → flag to user, don't delete

**Credentials and secrets** — mandatory check before any commit:

```bash
# Tokens, keys, passwords in documentation and source files
grep -rn "Bearer \|sk-\|password\|secret\|token\|api.key\|JWT_SECRET\|SMTP_PASSWORD" --include="*.md" --include="*.go" <dir>/ | grep -vi "variable\|example\|documentation\|placeholder\|change-me" | grep -v "_test.go"
```

A secret found outside test contexts is an emergency:
1. Remove immediately from the file
2. Flag to user — the secret is in git history, rotation needed
3. Do NOT continue the audit before user confirmation

### Phase 6 — Regeneration and verification

```bash
./scripts/generate-index.sh
go test ./conformity-tests/ -v
```

If conformity tests fail, fix the annotations — not the test.

## Scope per invocation

The user can target:
- `audit pkg/auth` → single package
- `audit service/` → entire service
- `full audit` → whole monorepo (expect several hours)

For large scope, process one directory at a time. Don't load the entire INDEX.map.

## What you NEVER do

- Modify application code (logic, control flow, signatures)
- Add features or fix bugs
- Touch functional tests
- Create `.go` files
- Refactor code
- Add godoc comments beyond `CLAUDE:*` annotations
- Add a WARN on a function with no real trap

## What you CAN delete

- Non-standard `CLAUDE:` annotations — AFTER migrating useful information
- Duplicate annotations in the same file — keep the most complete
- CLAUDE.md content without information (empty phrases, code rephrasing)
- Credentials and secrets — immediate deletion, user notification

Absolute rule: never delete information without migration. If a previous Claude wrote something, it had a reason. Find it, migrate to the right place, then clean up.

When in doubt: flag, don't delete.
