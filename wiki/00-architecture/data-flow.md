# Query Execution Data Flow

This document traces the complete lifecycle of a query in NeuG.

## Query Lifecycle Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            QUERY LIFECYCLE                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Cypher Query                                                           │
│   "MATCH (n:person) RETURN n.name"                                       │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                    PARSING PHASE                                  │  │
│   │   Parser (ANTLR4) → ParsedExpression Tree                        │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                    BINDING PHASE                                  │  │
│   │   Binder → BoundStatement (schema-aware)                         │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                    PLANNING PHASE                                 │  │
│   │   Planner → LogicalPlan (operator tree)                          │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                 OPTIMIZATION PHASE                                │  │
│   │   Optimizer → Optimized LogicalPlan                              │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                 CONVERSION PHASE                                  │  │
│   │   GOptPlanner → PhysicalPlan (Protocol Buffers)                  │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                 EXECUTION PHASE                                   │  │
│   │   PlanParser → Pipeline → Operators → Context                    │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│        │                                                                 │
│        ▼                                                                 │
│   QueryResult                                                            │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Phase Details

### 1. Parsing Phase

**Location**: `src/compiler/parser/`

**Input**: Cypher query string

**Output**: `ParsedExpression` tree

**Process**:
1. ANTLR4 lexer tokenizes the query
2. Parser generates parse tree using Cypher grammar
3. Transform parse tree to `ParsedExpression` objects

**Example**:
```cypher
MATCH (n:person)-[r:knows]->(m:person)
WHERE n.age > 30
RETURN n.name, m.name
```

Parsed as:
```
ParsedQueryExpression
├── MatchClause
│   ├── Pattern: (n:person)-[r:knows]->(m:person)
│   └── Where: n.age > 30
└── ReturnClause
    └── Expressions: [n.name, m.name]
```

**Key Files**:
- `src/compiler/parser/antlr_parser/` - ANTLR4 grammar files
- `src/compiler/parser/transform/` - Parse tree transformation
- `src/compiler/parser/expression/` - Parsed expression classes

### 2. Binding Phase

**Location**: `src/compiler/binder/`

**Input**: `ParsedExpression` tree

**Output**: `BoundStatement`

**Process**:
1. Resolve variable names against schema
2. Bind property references to actual columns
3. Validate types and semantics
4. Create bound expressions

**Example Binding**:
```
Parsed: n.age > 30
    │
    ▼
Bound: PropertyExpression(n, "age", INT64) > LiteralExpression(30)
```

**Key Bindings**:
| Parsed | Bound |
|--------|-------|
| Node pattern | `NodeExpression` with `NodeTableCatalogEntry` |
| Relationship pattern | `RelExpression` with `GRelTableCatalogEntry` |
| Property access | `PropertyExpression` with column reference |
| Function call | `ScalarFunctionExpression` with function pointer |

**Key Files**:
- `src/compiler/binder/bind/` - Statement binding
- `src/compiler/binder/expression/` - Expression binding
- `src/compiler/binder/query/` - Query binding

### 3. Planning Phase

**Location**: `src/compiler/planner/`

**Input**: `BoundStatement`

**Output**: `LogicalPlan`

**Process**:
1. Generate logical operator tree
2. Determine join orders
3. Create scan, extend, filter, project operators

**Example Logical Plan**:
```
LogicalProjection [n.name, m.name]
└── LogicalFilter (n.age > 30)
    └── LogicalExtend [r:knows] (n → m)
        └── LogicalScanNodeTable [n:person]
```

**Key Operators**:
| Operator | Purpose |
|----------|---------|
| `LogicalScanNodeTable` | Scan vertices of a label |
| `LogicalExtend` | Expand edges from vertices |
| `LogicalGetV` | Get vertex from edge endpoint |
| `LogicalFilter` | Apply predicates |
| `LogicalProjection` | Project expressions |
| `LogicalHashJoin` | Join two streams |
| `LogicalAggregate` | Group and aggregate |

**Key Files**:
- `src/compiler/planner/operator/` - Logical operator definitions
- `src/compiler/planner/plan/` - Plan construction

### 4. Optimization Phase

**Location**: `src/compiler/optimizer/`

**Input**: `LogicalPlan`

**Output**: Optimized `LogicalPlan`

**Process**: Apply 15+ optimization rules in sequence

**Optimization Pipeline**:
```cpp
// From optimizer.cpp
1. RemoveFactorizationRewriter    // Clean up structure
2. CorrelatedSubqueryUnnestSolver // Unnest subqueries
3. RemoveUnnecessaryJoinOptimizer // Remove useless joins
4. FilterPushDownOptimizer        // Push filters down
5. ProjectionPushDownOptimizer    // Push projections down
6. LimitPushDownOptimizer         // Push limits down
7. TopKOptimizer                  // OrderBy+Limit → TopK
8. AggKeyDependencyOptimizer      // Optimize aggregates
9. FilterPushDownPattern          // Pattern-specific filters
10. RenameDependentVarOpt         // Rename variables
11. ExpandGetVFusion              // Fuse Expand+GetV
12. FlatJoinToExpandOptimizer     // Convert joins to extends
13. CommonPatternReuseOptimizer   // Reuse common patterns
14. UnionAliasMapOptimizer        // Optimize UNION
15. CardinalityUpdater            // Update cardinalities
16. ProjectJoinConditionOptimizer // Optimize join projections
```

**Example Optimization**:

Before:
```
LogicalGetV (m)
└── LogicalExtend [r:knows] (n → m)
    └── LogicalScanNodeTable [n:person]
```

After (ExpandGetVFusion):
```
LogicalExtend [r:knows] (n → m, VERTEX)
└── LogicalScanNodeTable [n:person]
```

### 5. Conversion Phase (GOpt)

**Location**: `src/compiler/gopt/`

**Input**: `LogicalPlan`

**Output**: `PhysicalPlan` (Protocol Buffers)

**Process**:
1. `GAliasManager` assigns integer IDs to aliases
2. `GQueryConvertor` converts operators to PB format
3. `GExprConverter` converts expressions to PB format
4. `GTypeConverter` converts types to PB format

**Example Physical Plan (PB)**:
```protobuf
physical_plan {
  plan {
    opr {
      scan {
        scan_opt: VERTEX
        params { tables { id: 0 } }  # person label
        alias { value: 0 }           # n alias
      }
    }
    meta_data {
      alias: 0
      type { graph_type { element_opt: VERTEX ... } }
    }
  }
  plan {
    opr {
      edge {
        expand_opt: VERTEX
        direction: OUT
        params { tables { id: 1 } }  # knows label
        v_tag { value: 0 }           # start from n
        alias { value: 1 }           # m alias
      }
    }
    meta_data { ... }
  }
  # ... more operators
}
```

**Key Files**:
- `src/compiler/gopt/g_query_converter.cpp` - Main conversion logic
- `src/compiler/gopt/g_expr_converter.cpp` - Expression conversion
- `src/compiler/gopt/g_type_converter.cpp` - Type conversion

### 6. Execution Phase

**Location**: `src/execution/`

**Input**: `PhysicalPlan` (Protocol Buffers)

**Output**: `QueryResult`

**Process**:
1. `PlanParser` parses PB into `Pipeline`
2. `Pipeline` executes operators in sequence
3. Each operator transforms `Context`
4. Results collected in final `Context` columns

**Execution Flow**:
```cpp
// Simplified execution
Pipeline pipeline = PlanParser::get().parse(plan);

Context ctx;
for (auto& op : pipeline.operators_) {
    op->Eval(graph, params, ctx, timer);
}

return QueryResult(std::move(ctx));
```

**Context Columns**:
```
Context
├── columns[0]: VertexColumn [vid_1, vid_2, ...]
├── columns[1]: ValueColumn<string> ["Alice", "Bob", ...]
└── tag_ids: [0, 1]  # alias IDs
```

**Key Files**:
- `src/execution/execute/pipeline.cc` - Pipeline execution
- `src/execution/execute/plan_parser.cc` - PB parsing
- `src/execution/execute/ops/` - Operator implementations

## Example: Complete Query Trace

**Query**:
```cypher
MATCH (n:person {id: 123})-[:knows]->(m:person)
RETURN n.name, m.name
```

### Step-by-Step Trace

| Phase | Input | Output |
|-------|-------|--------|
| Parse | Cypher string | `ParsedQueryExpression` |
| Bind | Parsed tree | `BoundMatchStatement` with bound nodes `n`, `m` and relationship `r` |
| Plan | Bound statement | `LogicalPlan` with Scan → Extend → Project |
| Optimize | Logical plan | Fused Expand+GetV, pushed predicates |
| Convert | Optimized plan | `PhysicalPlan` with Scan → Edge → Project operators |
| Execute | Physical plan | `QueryResult` with 2 string columns |

### Generated Operators

```
1. Scan (person, alias=0, idx_predicate: id=123)
   → Context: VertexColumn[n]

2. EdgeExpand (knows, OUT, alias=1)
   → Context: VertexColumn[n], VertexColumn[m]

3. Project [n.name, m.name]
   → Context: ValueColumn<string>, ValueColumn<string>
```

## Performance Considerations

### Query Plan Caching

NeuG caches compiled physical plans:

```cpp
// From query_processor.cpp
auto cached = global_query_cache_->lookup(query, mode);
if (cached) {
    return cached;  // Skip compilation
}
// ... compile and cache
```

### Cardinality Estimation

Optimizer uses cardinality estimates for join ordering:

```cpp
// From cardinality_updater.cpp
void CardinalityUpdater::rewrite(LogicalPlan* plan) {
    // Estimate row counts for each operator
    // Used for join order optimization
}
```

## References

- [Compilation Pipeline](../03-query-engine/compilation-pipeline.md)
- [GOpt Framework](../03-query-engine/gopt-framework.md)
- [Pipeline Execution](../04-execution-engine/pipeline.md)