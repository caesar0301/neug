# Architecture Overview

This section covers NeuG's high-level system architecture and design philosophy.

## Documents

| Document | Description |
|----------|-------------|
| [system-design.md](./system-design.md) | Layered architecture, component interactions, design philosophy |
| [dual-mode-architecture.md](./dual-mode-architecture.md) | Embedded (AP) vs Service (TP) mode comparison |
| [data-flow.md](./data-flow.md) | Query lifecycle from Cypher to results |

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Client APIs (Python, C++, Java)            │
├─────────────────────────────────────────────────────────────────┤
│  Embedded Mode                     │      Service Mode          │
│  (db.connect())                    │      (db.serve())          │
├─────────────────────────────────────────────────────────────────┤
│                        Query Processor                          │
├─────────────────────────────────────────────────────────────────┤
│  Compiler Pipeline: Parser → Binder → Optimizer → Planner       │
├─────────────────────────────────────────────────────────────────┤
│                      Execution Engine                            │
├─────────────────────────────────────────────────────────────────┤
│   Transaction Manager (Embedded: global lock / Service: MVCC)   │
├─────────────────────────────────────────────────────────────────┤
│                     Storage Layer                                │
│   CSR Format │ Graph Storage │ Data Loaders (CSV, Arrow)       │
└─────────────────────────────────────────────────────────────────┘
```

## Design Philosophy

NeuG follows the same design philosophy as DuckDB — but for graph databases:

- **Lightweight**: Single binary, minimal dependencies
- **Embeddable**: Import directly into Python applications
- **Flexible**: Switch between embedded and service modes
- **Performant**: Record-breaking LDBC SNB Interactive benchmark (80,000+ QPS)

## Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **Compiler** | `src/compiler/` | Query compilation pipeline |
| **Execution** | `src/execution/` | Physical operator execution |
| **Storage** | `src/storages/` | CSR-based graph storage |
| **Transaction** | `src/transaction/` | MVCC and WAL for ACID |
| **Server** | `src/server/` | BRPC HTTP server for TP mode |