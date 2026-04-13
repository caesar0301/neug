# Optimizer Rules

This document describes NeuG's query optimization rules.

## Optimization Pipeline

```cpp
// From optimizer.cpp
void Optimizer::optimize(LogicalPlan* plan) {
    // Phase 1: Structure cleanup
    RemoveFactorizationRewriter().rewrite(plan);
    CorrelatedSubqueryUnnestSolver().solve(plan->getLastOperator().get());
    RemoveUnnecessaryJoinOptimizer().rewrite(plan);

    // Phase 2: Predicate pushdown
    FilterPushDownOptimizer(context).rewrite(plan);
    ProjectionPushDownOptimizer(context).rewrite(plan);
    LimitPushDownOptimizer().rewrite(plan);

    // Phase 3: Graph-specific optimization
    TopKOptimizer().rewrite(plan);
    AggKeyDependencyOptimizer().rewrite(plan);
    FilterPushDownPattern().rewrite(plan);
    RenameDependentVarOpt().rewrite(plan);
    ExpandGetVFusion(context->getCatalog()).rewrite(plan);
    FlatJoinToExpandOptimizer().rewrite(plan);
    CommonPatternReuseOptimizer(context).rewrite(plan);

    // Phase 4: Final cleanup
    UnionAliasMapOptimizer().rewrite(plan);
    RemoveSubqueryAsJoin().rewrite(plan);
    CardinalityUpdater(estimator, transaction).rewrite(plan);
    ProjectJoinConditionOptimizer(context).rewrite(plan);
}
```

## Phase 1: Structure Cleanup

### RemoveFactorizationRewriter

Removes redundant factorization structures from the plan.

**Before**:
```
LogicalAliasMap
└── LogicalScanNodeTable
```

**After**:
```
LogicalScanNodeTable  (with merged aliases)
```

### CorrelatedSubqueryUnnestSolver

Unnests correlated subqueries into joins.

**Before**:
```
LogicalFilter (WHERE EXISTS { MATCH ... })
└── LogicalScanNodeTable
```

**After**:
```
LogicalHashJoin (SEMI)
├── LogicalScanNodeTable
└── LogicalScanNodeTable (subquery)
```

### RemoveUnnecessaryJoinOptimizer

Removes joins that don't affect results.

**Before**:
```
LogicalHashJoin (on constant)
├── LogicalScanNodeTable
└── LogicalScanNodeTable (unused)
```

**After**:
```
LogicalScanNodeTable
```

## Phase 2: Predicate Pushdown

### FilterPushDownOptimizer

Pushes filter predicates as close to data sources as possible.

**Before**:
```
LogicalFilter (n.age > 30)
└── LogicalExtend
    └── LogicalScanNodeTable
```

**After**:
```
LogicalExtend
└── LogicalScanNodeTable (predicate: age > 30)
```

### ProjectionPushDownOptimizer

Pushes projections to reduce intermediate data.

**Before**:
```
LogicalProjection [n.name]
└── LogicalExtend
    └── LogicalScanNodeTable (all columns)
```

**After**:
```
LogicalProjection [n.name]
└── LogicalExtend
    └── LogicalScanNodeTable (only name column)
```

### LimitPushDownOptimizer

Pushes LIMIT operators to reduce data early.

**Before**:
```
LogicalLimit (10)
└── LogicalProjection
    └── LogicalExtend
        └── LogicalScanNodeTable
```

**After**:
```
LogicalLimit (10)
└── LogicalProjection
    └── LogicalLimit (10)  // Pushed down
        └── LogicalExtend
            └── LogicalScanNodeTable
```

## Phase 3: Graph-Specific Optimization

### TopKOptimizer

Combines ORDER BY + LIMIT into efficient TopK operation.

**Before**:
```
LogicalLimit (10)
└── LogicalOrderBy (name ASC)
    └── LogicalScanNodeTable
```

**After**:
```
LogicalTopK (10, name ASC)
└── LogicalScanNodeTable
```

### ExpandGetVFusion

Fuses EdgeExpand + GetV into a single operator.

**Before**:
```
LogicalGetV (m)
└── LogicalExtend [r:knows] (n → m)
    └── LogicalScanNodeTable [n]
```

**After**:
```
LogicalExtend [r:knows] (n → m, VERTEX)  // Fused
└── LogicalScanNodeTable [n]
```

**Conditions for fusion**:
- No predicate filtering on intermediate edge
- No label filtering on destination vertex
- Result option allows fusion

### FlatJoinToExtendOptimizer

Converts flat joins to graph extends when possible.

**Before**:
```
LogicalHashJoin (n.id = r.src_id)
├── LogicalScanNodeTable [n:person]
└── LogicalScanNodeTable [r:knows]
```

**After**:
```
LogicalExtend [r:knows]
└── LogicalScanNodeTable [n:person]
```

### CommonPatternReuseOptimizer

Identifies and reuses common subpatterns.

**Query**:
```cypher
MATCH (a)-[:knows]->(b), (a)-[:knows]->(c)
```

**Before** (two separate extends):
```
LogicalHashJoin
├── LogicalExtend [knows] (a → b)
│   └── LogicalScanNodeTable [a]
└── LogicalExtend [knows] (a → c)
    └── LogicalScanNodeTable [a]
```

**After** (reuse):
```
LogicalHashJoin
├── LogicalGetV (b)
│   └── LogicalExtend [knows] (a → *)
├── LogicalGetV (c)
│   └── LogicalExtend [knows] (a → *)  // Reuses same extend
└── LogicalScanNodeTable [a]
```

## Phase 4: Final Cleanup

### UnionAliasMapOptimizer

Optimizes UNION alias handling.

### CardinalityUpdater

Updates cardinality estimates for cost-based decisions.

```cpp
void CardinalityUpdater::rewrite(LogicalPlan* plan) {
    auto root = plan->getLastOperator();
    updateCardinality(root.get());
}

void CardinalityUpdater::updateCardinality(LogicalOperator* op) {
    for (auto& child : op->getChildren()) {
        updateCardinality(child.get());
    }

    switch (op->getOperatorType()) {
        case LogicalOperatorType::SCAN_NODE_TABLE: {
            auto scan = op->ptrCast<LogicalScanNodeTable>();
            cardinality = estimator.estimateScan(scan->getTableIDs());
            break;
        }
        case LogicalOperatorType::EXTEND: {
            auto extend = op->ptrCast<LogicalExtend>();
            cardinality = child_cardinality * estimator.estimateDegree(extend->getLabelIds());
            break;
        }
        // ...
    }
}
```

### ProjectJoinConditionOptimizer

Optimizes join condition projections.

## Rule Implementation Pattern

```cpp
class MyOptimizer : public LogicalOperatorVisitor {
public:
    void rewrite(LogicalPlan* plan) {
        auto root = plan->getLastOperator();
        auto newRoot = visitOperator(root);
        plan->setLastOperator(newRoot);
    }

private:
    std::shared_ptr<LogicalOperator> visitOperator(
        const std::shared_ptr<LogicalOperator>& op) {

        // Bottom-up traversal
        for (auto i = 0u; i < op->getNumChildren(); ++i) {
            op->setChild(i, visitOperator(op->getChild(i)));
        }

        // Apply transformation
        return visitOperatorReplaceSwitch(op);
    }

    std::shared_ptr<LogicalOperator> visitOperatorReplaceSwitch(
        const std::shared_ptr<LogicalOperator>& op) {

        switch (op->getOperatorType()) {
            case LogicalOperatorType::FILTER:
                return visitFilterReplace(op);
            case LogicalOperatorType::EXTEND:
                return visitExtendReplace(op);
            // ...
            default:
                return op;
        }
    }

    std::shared_ptr<LogicalOperator> visitFilterReplace(
        const std::shared_ptr<LogicalOperator>& op) {
        auto filter = op->ptrCast<LogicalFilter>();

        // Check if we can push predicate down
        auto child = filter->getChild(0);
        if (canPushDown(filter->getPredicate(), child)) {
            // Push predicate into child
            pushDownPredicate(filter->getPredicate(), child);
            return child;  // Remove filter
        }

        return op;
    }
};
```

## Performance Impact

| Rule | Typical Improvement |
|------|---------------------|
| FilterPushDown | 2-10x for selective queries |
| ProjectionPushDown | 1.5-3x for wide tables |
| ExpandGetVFusion | 1.5-2x for traversal queries |
| TopKOptimizer | 10-100x for ORDER BY + LIMIT |
| LimitPushDown | 2-5x for limit queries |

## References

- [Compilation Pipeline](./compilation-pipeline.md)
- [Logical Operators](./logical-operators.md)
- [GOpt Framework](./gopt-framework.md)