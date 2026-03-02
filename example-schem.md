# Example: Package-level technical schema

> Anonymized from a real service router package (14 files, 1800+ lines of Go).
> An agent reads this instead of opening the source files.

```
╔════════════════════════════════════════════════════════════════════════════════╗
║  router -- Smart service router: local or remote dispatch via SQLite         ║
╠════════════════════════════════════════════════════════════════════════════════╣
║  Module: shared-lib/router                                                   ║
║  Files:  router.go, schema.go, watcher.go, admin.go, breaker.go,           ║
║          retry.go, middleware.go, factory_http.go, factory_mcp.go,          ║
║          observe.go, inspect.go, fallback.go, gateway.go, errors.go        ║
║  Deps:   shared-lib/dbopen, shared-lib/transport, shared-lib/sanitize      ║
╚════════════════════════════════════════════════════════════════════════════════╝

ARCHITECTURE -- "Job as Library" pattern
=========================================

  ┌────────────┐     ┌──────────────────────────────────────────────────┐
  │  Caller    │     │                   Router                         │
  │            │     │                                                  │
  │ Call(ctx,  ├────>│  1. noop?   -> return nil, nil                   │
  │  "billing",│     │  2. remote? -> remoteEntries[svc].handler(...)   │
  │  payload)  │     │  3. local?  -> localHandlers[svc](ctx, payload)  │
  │            │     │  4. none?   -> ErrServiceNotFound                │
  └────────────┘     └──────────────────────────────────────────────────┘
                                        ^
                                        │ Reload() reads routes table
                                        │
                     ┌──────────────────────────────────────────────────┐
                     │           SQLite routes table                     │
                     │  ┌────────────┬──────────┬───────────┬──────────┐│
                     │  │service_name│ strategy │ endpoint  │ config   ││
                     │  │ TEXT (PK)  │ TEXT     │ TEXT      │ TEXT/JSON││
                     │  ├────────────┼──────────┼───────────┼──────────┤│
                     │  │ billing    │ local    │           │ {}       ││
                     │  │ search     │ quic     │ 10.0.0.5  │ {..}    ││
                     │  │ embed      │ http     │ http://.. │ {..}    ││
                     │  │ disabled   │ noop     │           │ {}       ││
                     │  └────────────┴──────────┴───────────┴──────────┘│
                     └──────────────────────────────────────────────────┘

HOT-RELOAD LOOP
================

  ┌─────────────────┐   PRAGMA data_version   ┌──────────────┐
  │  Watch(ctx, db, ├──────── poll 200ms ────> │  SQLite DB   │
  │  200ms)         │                          └──────┬───────┘
  └────────┬────────┘                                 │
           │  version changed?                        │
           │  YES -> Reload(ctx, db)                  │
           │         - diff fingerprints              │
           │         - close stale handlers           │
           │         - build new via factories        │
           └──────────────────────────────────────────┘

  fingerprint = strategy + "|" + endpoint + "|" + config
  Only changed routes are rebuilt. Unchanged routes keep their connections.

DATABASE SCHEMA
================

  CREATE TABLE IF NOT EXISTS routes (
      service_name TEXT PRIMARY KEY,
      strategy     TEXT NOT NULL CHECK(strategy IN
                       ('local','quic','http','mcp','dbsync','embed','noop')),
      endpoint     TEXT,
      config       TEXT DEFAULT '{}',
      updated_at   INTEGER NOT NULL DEFAULT (strftime('%s','now'))
  );

TRANSPORT FACTORIES
====================

  ┌────────────────────────────────────────────────────────────────────┐
  │  RegisterTransport("http",  HTTPFactory())                        │
  │  RegisterTransport("mcp",   MCPFactory())                         │
  │  RegisterTransport("quic",  ...)  // user-provided                │
  │  RegisterTransport("embed", EmbedFactory())                       │
  └────────────────────────────────────────────────────────────────────┘

  HTTPFactory:
    endpoint  -> POST payload to URL
    SSRF guard via sanitize.ValidateURL
    Response cap: 10 MiB
    close()   -> client.CloseIdleConnections()

  MCPFactory:
    endpoint  -> QUIC address "ip:port"
    Eager connect on Reload (fail fast)
    Payload unmarshalled as JSON map -> CallTool(ctx, toolName, args)
    close()   -> client.Close()

GATEWAY
========

  ┌──────────────────┐    POST /{service}     ┌─────────────────────┐
  │  Remote caller   │ ──────────────────────> │  router.Gateway()   │
  │  (HTTPFactory)   │                         │  http.Handler       │
  └──────────────────┘                         │                     │
                                               │  dispatch to LOCAL  │
                                               │  handlers ONLY      │
                                               │  (no remote->remote)│
                                               │  Body cap: 16 MiB   │
                                               └─────────────────────┘

MIDDLEWARE CHAIN
=================

  Handler = func(ctx, []byte) ([]byte, error)
  HandlerMiddleware = func(Handler) Handler

  ┌─────────────────────────────────────────────────────────────────┐
  │  Chain(mws...) -- compose left-to-right                        │
  │                                                                 │
  │  Logging(logger)                  -- log duration + error       │
  │  Timeout(d)                       -- context.WithTimeout        │
  │  Recovery(logger)                 -- panic -> ErrPanic          │
  │  WithRetry(max, backoff, logger)  -- exp backoff, skip circuit  │
  │  WithCircuitBreaker(cb, svc)      -- ErrCircuitOpen on open     │
  │  WithFallback(local, svc, logger) -- remote fail -> try local   │
  └─────────────────────────────────────────────────────────────────┘

CIRCUIT BREAKER
================

  States: Closed(0) -> Open(1) -> HalfOpen(2)
  Defaults: threshold=5, resetTimeout=30s, halfOpenMax=2

CORE TYPES
===========

  Handler            func(ctx, []byte) ([]byte, error)
  TransportFactory   func(endpoint, config) (Handler, close func(), error)
  HandlerMiddleware  func(Handler) Handler
  Router             { localHandlers, remoteEntries, routeSnap, factories }
  Admin              { db *sql.DB }
  CircuitBreaker     { state, failures, successes, threshold, ... }

KEY FUNCTIONS
==============

  New(opts...) *Router
  (r) RegisterLocal(service, Handler)
  (r) RegisterTransport(protocol, TransportFactory)
  (r) Call(ctx, service, payload) ([]byte, error)
  (r) Reload(ctx, db) error
  (r) Watch(ctx, db, interval)          -- blocking, run in goroutine
  (r) Close() error
  (r) Gateway() http.Handler
  OpenDB(path) (*sql.DB, error)
  NewAdmin(db) *Admin
  HTTPFactory(opts...) TransportFactory
  MCPFactory() TransportFactory
  Chain(mws...) HandlerMiddleware
```

> This schema replaces reading 14 source files (1800+ lines).
> An agent running `cat router_schem.md` gets the full architecture in one read.
