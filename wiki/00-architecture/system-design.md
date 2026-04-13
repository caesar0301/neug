# System Design

This document provides a comprehensive overview of NeuG's system architecture.

## Design Philosophy

NeuG follows the same design philosophy as DuckDB — but for graph databases:

| Principle | Description |
|-----------|-------------|
| **Lightweight** | Single binary, minimal external dependencies |
| **Embeddable** | Import directly into Python applications |
| **Flexible** | Switch between embedded and service modes |
| **Performant** | Optimized for both analytical and transactional workloads |

## Layered Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Client Layer                                  │
│  Python API  │  Java Driver  │  C++ API  │  HTTP/REST               │
├─────────────────────────────────────────────────────────────────────┤
│                        Session Layer                                 │
│  Connection  │  Session  │  ConnectionManager  │  SessionPool       │
├─────────────────────────────────────────────────────────────────────┤
│                        Query Layer                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  Parser  │→│  Binder  │→│ Optimizer │→│ Planner  │            │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘            │
├─────────────────────────────────────────────────────────────────────┤
│                      Execution Layer                                 │
│  Pipeline  │  Operators  │  Context  │  Expression Evaluator       │
├─────────────────────────────────────────────────────────────────────┤
│                    Transaction Layer                                 │
│  ReadTx  │  InsertTx  │  UpdateTx  │  MVCC  │  WAL                 │
├─────────────────────────────────────────────────────────────────────┤
│                       Storage Layer                                  │
│  PropertyGraph  │  VertexTable  │  EdgeTable  │  CSR               │
├─────────────────────────────────────────────────────────────────────┤
│                       Utility Layer                                  │
│  mmap_array  │  IdIndexer  │  Serialization  │  ThreadLocal        │
└─────────────────────────────────────────────────────────────────────┘
```

## Component Overview

### Client Layer

| Component | Location | Purpose |
|-----------|----------|---------|
| Python API | `tools/python_bind/` | Primary user interface |
| Java Driver | `tools/java_driver/` | JVM integration |
| C++ API | `include/neug/main/` | Native C++ interface |
| HTTP/REST | `src/server/` | Network access |

### Session Layer

| Component | Location | Purpose |
|-----------|----------|---------|
| Connection | `src/main/connection.cc` | Embedded mode connection |
| Session | `tools/python_bind/neug/session.py` | Remote session (TP mode) |
| ConnectionManager | `src/main/connection_manager.cc` | Connection pooling |
| SessionPool | `src/server/session_pool.cc` | Server-side session pooling |

### Query Layer

| Component | Location | Purpose |
|-----------|----------|---------|
| Parser | `src/compiler/parser/` | ANTLR4-based Cypher parsing |
| Binder | `src/compiler/binder/` | Semantic analysis, schema binding |
| Optimizer | `src/compiler/optimizer/` | Rule-based query optimization |
| Planner | `src/compiler/planner/` | Logical/physical plan generation |
| GOpt | `src/compiler/gopt/` | Graph optimization framework |

### Execution Layer

| Component | Location | Purpose |
|-----------|----------|---------|
| Pipeline | `src/execution/execute/pipeline.cc` | Operator orchestration |
| Operators | `src/execution/execute/ops/` | Physical operator implementations |
| Context | `src/execution/common/context.cc` | Execution state management |
| Expression | `src/execution/expression/` | Expression evaluation |

### Transaction Layer

| Component | Location | Purpose |
|-----------|----------|---------|
| ReadTransaction | `src/transaction/read_transaction.cc` | Read-only snapshot queries |
| InsertTransaction | `src/transaction/insert_transaction.cc` | Bulk insert operations |
| UpdateTransaction | `src/transaction/update_transaction.cc` | Full CRUD + DDL |
| VersionManager | `src/transaction/version_manager.cc` | MVCC timestamp management |
| WAL | `src/transaction/wal/` | Write-ahead logging |

### Storage Layer

| Component | Location | Purpose |
|-----------|----------|---------|
| PropertyGraph | `src/storages/graph/property_graph.cc` | Main storage engine |
| VertexTable | `src/storages/graph/vertex_table.cc` | Vertex storage per label |
| EdgeTable | `src/storages/graph/edge_table.cc` | Edge storage with CSR |
| CSR | `src/storages/csr/` | Compressed Sparse Row format |

## Data Flow

### Query Execution Flow

```
User Query (Cypher)
       │
       ▼
┌─────────────────┐
│  QueryProcessor │  Entry point
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Parser      │  ANTLR4 → ParsedExpression tree
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Binder      │  Bind to schema → BoundStatement
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Planner      │  Generate LogicalPlan
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Optimizer    │  Apply 15+ optimization rules
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   GOptPlanner   │  Convert to PhysicalPlan (PB)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   PlanParser    │  Parse PB → Pipeline
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Pipeline     │  Execute operators
└────────┬────────┘
         │
         ▼
   QueryResult
```

### Transaction Flow

```
Connection.execute(query, mode)
            │
            ▼
    ┌───────────────┐
    │  Access Mode  │
    └───────┬───────┘
            │
    ┌───────┼───────┐
    │       │       │
    ▼       ▼       ▼
  read   insert   update
    │       │       │
    ▼       ▼       ▼
┌───────┐┌───────┐┌───────┐
│ReadTx ││InsertTx││UpdateTx│
└───┬───┘└───┬───┘└───┬───┘
    │        │        │
    ▼        ▼        ▼
┌─────────────────────────┐
│    VersionManager       │
│  (Timestamp Allocation) │
└─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│     PropertyGraph       │
│   (Storage Interface)   │
└─────────────────────────┘
```

## Key Design Decisions

### 1. Dual-Mode Architecture

NeuG supports two operational modes:

| Mode | Lock Model | Use Case |
|------|------------|----------|
| **Embedded (AP)** | Global lock | Analytics, batch processing |
| **Service (TP)** | MVCC | Real-time, concurrent access |

### 2. CSR Storage Format

Compressed Sparse Row format chosen for:
- Space efficiency: O(V + E)
- Cache-friendly neighbor iteration
- Efficient edge traversal operations

### 3. Protocol Buffers IR

Physical plans serialized as Protocol Buffers:
- Language-neutral representation
- Efficient serialization
- Enables future distributed execution

### 4. Columnar Execution

Context uses columnar data layout:
- Vectorized operations
- Better cache utilization
- Compatible with Apache Arrow

## Thread Safety

| Component | Thread Safety |
|-----------|---------------|
| PropertyGraph | Read-only concurrent access |
| VertexTable | Read concurrent, write exclusive |
| EdgeTable | Read concurrent, write per-vertex lock |
| Connection | Not thread-safe (create per thread) |
| Session | Not thread-safe (create per thread) |

## Memory Management

| Level | Configuration | Behavior |
|-------|---------------|----------|
| `kLowMemory` | Minimal RAM | Aggressive disk usage |
| `kMediumMemory` | Balanced | Moderate caching |
| `kHighMemory` | Maximum RAM | Full in-memory when possible |

## References

- [Dual-Mode Architecture](./dual-mode-architecture.md)
- [Data Flow](./data-flow.md)
- [GOpt Framework](../03-query-engine/gopt-framework.md)