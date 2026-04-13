# Query Engine Overview

This section documents NeuG's query compilation and optimization pipeline.

## Documents

| Document | Description |
|----------|-------------|
| [compilation-pipeline.md](./compilation-pipeline.md) | Parser → Binder → Optimizer → Planner |
| [gopt-framework.md](./gopt-framework.md) | GOpt optimization framework architecture |
| [optimizer-rules.md](./optimizer-rules.md) | 15+ optimization rules |
| [logical-operators.md](./logical-operators.md) | Logical plan operator definitions |
| [physical-operators.md](./physical-operators.md) | Physical plan operator definitions |
| [expression-evaluation.md](./expression-evaluation.md) | Expression conversion and evaluation |

## Compilation Pipeline

```
Cypher Query
    │
    ▼
┌─────────────┐
│   Parser    │  ANTLR4-based Cypher parser
│ (antlr4/)   │  Produces ParsedExpression tree
└─────────────┘
    │
    ▼
┌─────────────┐
│   Binder    │  Semantic analysis, schema binding
│ (binder/)   │  Produces BoundStatement
└─────────────┘
    │
    ▼
┌─────────────┐
│  Planner    │  Logical plan generation
│ (planner/)  │  Produces LogicalPlan
└─────────────┘
    │
    ▼
┌─────────────┐
│  Optimizer  │  Rule-based optimization
│(optimizer/) │  15+ optimization rules
└─────────────┘
    │
    ▼
┌─────────────┐
│ GOptPlanner │  Logical → Physical conversion
│  (gopt/)    │  Produces PhysicalPlan (Protocol Buffers)
└─────────────┘
```

## GOpt Framework

GOpt is NeuG's graph optimization framework, adapted from GraphScope:

- **GCatalog**: Schema and function management
- **GQueryConvertor**: Logical plan to physical plan conversion
- **GExprConverter**: Expression to Protocol Buffers conversion
- **GAliasManager**: Query variable to ID mapping

## Optimization Rules

| Phase | Rules |
|-------|-------|
| Structure cleanup | RemoveFactorization, RemoveUnnecessaryJoin |
| Predicate pushdown | FilterPushDown, ProjectionPushDown, LimitPushDown |
| Graph-specific | ExpandGetVFusion, FlatJoinToExpand, CommonPatternReuse |
| Final cleanup | UnionAliasMap, CardinalityUpdater |

## Key Directories

| Directory | Purpose |
|-----------|---------|
| `src/compiler/parser/` | ANTLR4 grammar and parsing |
| `src/compiler/binder/` | Semantic analysis and binding |
| `src/compiler/planner/` | Logical plan generation |
| `src/compiler/optimizer/` | Query optimization rules |
| `src/compiler/gopt/` | GOpt framework implementation |