# Execution Engine Overview

This section documents NeuG's runtime execution engine.

## Documents

| Document | Description |
|----------|-------------|
| [pipeline.md](./pipeline.md) | Pipeline execution model |
| [context-columns.md](./context-columns.md) | Columnar execution context |
| [plan-parser.md](./plan-parser.md) | Physical plan to Pipeline conversion |
| [operator-implementations.md](./operator-implementations.md) | Scan, Edge, Path, Join, etc. |

## Execution Architecture

```
PhysicalPlan (Protocol Buffers)
        │
        ▼
┌─────────────┐
│ PlanParser  │  Parses PB into operator sequence
└─────────────┘
        │
        ▼
┌─────────────┐
│  Pipeline   │  Orchestrates operator execution
└─────────────┘
        │
        ▼
┌─────────────┐
│  Context    │  Columnar data during execution
└─────────────┘
```

## Key Classes

| Class | Header | Purpose |
|-------|--------|---------|
| `Pipeline` | `execution/execute/pipeline.h` | Operator orchestration |
| `IOperator` | `execution/execute/operator.h` | Base operator interface |
| `Context` | `execution/common/context.h` | Execution state and columns |
| `PlanParser` | `execution/execute/plan_parser.h` | PB → Pipeline conversion |

## Operator Categories

| Category | Directory | Examples |
|----------|-----------|----------|
| **Retrieve** | `ops/retrieve/` | Scan, Edge, Path, Project, Join, GroupBy |
| **Insert** | `ops/insert/` | CreateVertex, CreateEdge |
| **Batch** | `ops/batch/` | BatchInsert, BatchDelete, DataSource, DataExport |
| **DDL** | `ops/ddl/` | CreateVertexType, CreateEdgeType, Drop |
| **Admin** | `ops/admin/` | Checkpoint, Extension |

## Column Types

| Type | Purpose |
|------|---------|
| `VertexColumn` | Vertex IDs |
| `EdgeColumn` | Edge references |
| `PathColumn` | Path data |
| `ValueColumn` | Primitive values |
| `ListColumn` | Collections |
| `StructColumn` | Nested structures |

## Execution Flow

1. **PlanParser** converts PhysicalPlan PB to Pipeline
2. **Pipeline.Execute()** iterates through operators
3. Each **IOperator.Eval()** transforms Context
4. Results collected in final Context columns