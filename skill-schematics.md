---
name: doc-schematics
description: Generate and maintain ASCII technical schemas (*_schem.md) for a project, service, or package. Run in a dedicated session. Produces *_schem.md files. Never touches application code or CLAUDE.md files.
---

A schema is a map. Not a manual, not exhaustive documentation, not a code summary.

A schema answers five questions:
1. What is it? (one sentence + metadata)
2. What is it made of? (file tree)
3. How do the parts connect? (architecture diagram)
4. What data persists? (SQL schemas / structures)
5. What goes in and out? (data flow)

Everything else belongs in the CLAUDE.md (rules, traps, decisions) or the code (implementation).

## Schema vs CLAUDE.md boundary

| Schema | CLAUDE.md |
|--------|-----------|
| SQL DDL (tables, columns, types) | Why `_txlock=immediate` |
| Architecture diagram (who calls who) | Trap: swapIndex without cache.clear |
| Data flow (input → transform → output) | Service startup order |
| Public types (names, fields) | What a developer would break unknowingly |
| Config with defaults | Why this default and not another |
| Memory/storage size per unit | Contextual benchmark results |

Rule: if the information changes when you modify a **structure** (table, type, file, connection), it's the schema. If it changes when you modify a **behavior** (logic, concurrency, deployment), it's the CLAUDE.md.

## Anatomy of a schema

Each schema follows this order. Empty sections are omitted, not left as placeholders.

### 1. Header (required)

```markdown
# projectname — Technical Schema

**One sentence. What it is, not how it works.**

Module: `github.com/org/project`
Go: 1.25 | External deps: list | CGO_ENABLED=0
Binaries: `cmd/x`, `cmd/y` — or "library-only, no cmd/"
```

Three lines max after the title.

### 2. File tree (required)

```
project/
├── file.go           Role in 5-8 words
├── other.go          Role in 5-8 words
├── internal/
│   ├── store/        SQLite persistence
│   └── handlers/     HTTP handlers
├── cmd/
│   └── project/
│       └── main.go   Entry point
└── schema.sql        Full DDL
```

No `go.mod`, `go.sum`, `LICENSE`, `README`. Only files containing logic.

### 3. Architecture diagram (required)

The central block. Shows components and their connections.

```
╔══════════════════╗         ╔══════════════════╗
║   Component A    ║         ║   Component B    ║
║                  ║         ║                  ║
║  key fields      ║────────▶║  key fields      ║
║  of struct       ║         ║  of struct       ║
╚══════════════════╝         ╚══════════════════╝
         │
         ▼
╔══════════════════╗
║   Component C    ║
╚══════════════════╝
```

### 4. Data schema (if persistence)

```
╔══════════════════════════════════════════════════╗
║  TABLE: table_name                               ║
╠═══════════════╤═══════════╤══════════════════════╣
║  Column       │ Type      │ Description          ║
╠═══════════════╪═══════════╪══════════════════════╣
║  id           │ INTEGER   │ PRIMARY KEY          ║
║  name         │ TEXT      │ NOT NULL UNIQUE       ║
╚═══════════════╧═══════════╧══════════════════════╝
```

Every table from the SQL schema must appear. Columns, types, constraints.

### 5. Data flow (if the project transforms data)

```
Input (source)
  │
  ▼
┌──────────┐    call/message    ┌──────────┐
│ Step 1   │ ──────────────────▶ │ Step 2   │
│ (verb)   │                     │ (verb)   │
└──────────┘                     └──────────┘
  │
  ▼
Output (destination)
```

One flow per main operation. Build, Search, Insert — not a single flow mixing everything.

### 6. Public types (if library)

```
╔═══════════════════════════════════════════════╗
║  type Config struct {                         ║
║      Field1  int     // role (default: 64)    ║
║      Field2  float64 // role (default: 0.05)  ║
║  }                                            ║
╠═══════════════════════════════════════════════╣
║  type Result struct {                         ║
║      ID    []byte                             ║
║      Score float64                            ║
║  }                                            ║
╚═══════════════════════════════════════════════╝
```

Only exported types used by consumers. Not internal types.

### 7. Consumers (if library)

Who imports this package, how, and in what mode.

### 8. Dependency matrix (if multi-package)

```
  ┌─────────┬─────┬─────┬─────┐
  │ pkg     │ svc1│ svc2│ svc3│
  ├─────────┼─────┼─────┼─────┤
  │ auth    │  ●  │  ●  │     │
  │ dbopen  │  ●  │  ●  │  ●  │
  └─────────┴─────┴─────┴─────┘
```

Only for ecosystem schemas or shared libraries.

## ASCII conventions

### Characters

```
Thick frames (major components) : ╔ ═ ╗ ║ ╚ ╝ ╠ ╣ ╬ ╤ ╧ ╪
Thin frames (sub-components)    : ┌ ─ ┐ │ └ ┘ ├ ┤ ┼ ┬ ┴
Data arrows                     : ──▶  ◀──  ──→  ←──
Heavy flow arrows               : ══▶  ◀══
Vertical                        : │ ▼ ▲
Tree                            : ├── └──
Bullet in table                 : ●
```

### Alignment rules

- Max width per block: 80 characters (terminals truncate beyond)
- All blocks at the same level have the same width
- Table columns are aligned — verify by counting characters
- Horizontal arrows sit on the same line as the middle of the source block
- Never leave an empty block

### Alignment verification

After each diagram, count the characters of the first and last line of each block. They must be identical. If a line's content is shorter, pad with spaces.

## Production sequence

### New schema

1. `cat <dir>/CLAUDE.md` — understand role and dependencies
2. `grep -rn "CLAUDE:SUMMARY" --include="*.go" <dir>/` — file inventory
3. `grep -rn "CLAUDE:EXPORTS" --include="*.go" <dir>/` — public API
4. `grep -rn "CLAUDE:DEPENDS" --include="*.go" <dir>/` — dependency graph
5. If persistence: `cat <dir>/schema.sql` or `grep -n "CREATE TABLE" --include="*.go" <dir>/`
6. Write the schema section by section, in the order defined above
7. Verify each assertion (verification phase below)

### Updating existing schema

1. `grep -rn "CLAUDE:SUMMARY" --include="*.go" <dir>/` — compare with schema's file tree
2. Files added/removed? → update section 2
3. Public types changed? `grep -rn "CLAUDE:EXPORTS"` → update section 6
4. SQL tables changed? → update section 4
5. New connections between components? → update section 3
6. Verify ASCII alignment of each modified block

## Verification

Every fact in the schema must be verifiable by grep. Before finalizing:

```bash
# Every file listed in the tree exists
ls <dir>/file.go

# Every table listed exists in the DDL
grep -c "CREATE TABLE table_name" <dir>/schema.sql

# Every public type listed is actually exported
grep -n "^type TypeName struct" --include="*.go" <dir>/

# Every arrow A → B corresponds to a real import or call
grep -rn "import.*packageB\|packageB\." --include="*.go" <dir_of_A>/

# Every table column matches the actual DDL
# (read the CREATE TABLE and compare column by column)
```

A schema with an arrow that doesn't correspond to any import is worse than a schema without the arrow. Don't invent. If the connection isn't verifiable, don't draw it.

## Size targets

| Scope | Lines | Tokens (~) |
|-------|-------|-----------|
| Sub-package (dbopen, vtq) | 50-100 | 1-2K |
| Library (vector search, tenant pool) | 200-350 | 5-8K |
| Service (web app, forum) | 250-400 | 6-10K |
| Ecosystem | 150-250 | 4-6K |

If the schema exceeds the upper target, it probably contains content that belongs in the CLAUDE.md or the code.

## Ecosystem schema

An ecosystem schema does not detail the internal components of each service. It shows:

1. Overview — all services as blocks, their connections
2. Main data flow — from user input to output
3. Secondary flows — replication, auth, observability
4. Dependency matrix — who uses what in the shared packages
5. Network topology — ports, protocols, servers
6. Database inventory — service, file, mode (RW/RO)

Each block links to the service's detailed schema: `see service/service_schem.md`.

## Naming

The file is named `{directory_name}_schem.md` and lives at the root of the directory it documents.

```
mylib/mylib_schem.md
service/service_schem.md
shared-lib/auth/auth_schem.md
```

Non-negotiable convention: schema name matches directory name. No aliases, no renaming.

## What you NEVER do

- Invent a connection not verifiable by grep
- Document behavior (logic, concurrency, traps) — that's the CLAUDE.md
- Exceed 80 characters wide in an ASCII block
- Leave an empty block or placeholder section
- Copy source code into the schema
- Include benchmark results (they change, the schema doesn't)
- Include architecture decisions (why X — that's the CLAUDE.md)
- Modify application code, CLAUDE.md files, or annotations
