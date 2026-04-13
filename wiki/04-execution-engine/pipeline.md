# Pipeline Execution

This document describes NeuG's pipeline execution model.

## Overview

NeuG uses a **pipeline execution model** for query processing:

1. Physical plan (Protocol Buffers) is parsed into a `Pipeline`
2. Pipeline contains a sequence of `IOperator` objects
3. Operators are executed in order, transforming `Context`
4. Results are collected in the final `Context`

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Pipeline Execution                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   PhysicalPlan (PB)                                             │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────┐                                              │
│   │ PlanParser  │  Parse PB into Pipeline                      │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                      Pipeline                            │  │
│   │                                                         │  │
│   │   ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐  ┌─────┐  │  │
│   │   │ Scan  │→│ Edge  │→│Filter │→│Project│→│Sink │  │  │
│   │   └───────┘  └───────┘  └───────┘  └───────┘  └─────┘  │  │
│   │                                                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │   Context   │  Columnar execution state                    │
│   └─────────────┘                                              │
│       │                                                         │
│       ▼                                                         │
│   QueryResult                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Pipeline Class

### Definition

```cpp
// From execute/pipeline.h
class Pipeline {
public:
    Pipeline() = default;

    // Execute the pipeline
    void Execute(IStorageInterface& storage,
                 Context&& ctx,
                 ParamsMap& params,
                 OprTimer* timer = nullptr);

    // Add operator
    void AddOperator(std::unique_ptr<IOperator> op) {
        operators_.push_back(std::move(op));
    }

    // Get operators
    const std::vector<std::unique_ptr<IOperator>>& GetOperators() const {
        return operators_;
    }

private:
    std::vector<std::unique_ptr<IOperator>> operators_;
};
```

### Execution

```cpp
void Pipeline::Execute(IStorageInterface& storage,
                        Context&& ctx,
                        ParamsMap& params,
                        OprTimer* timer) {
    // Execute operators in sequence
    for (auto& op : operators_) {
        if (timer) timer->start(op->get_operator_name());

        op->Eval(storage, params, std::move(ctx), timer);

        if (timer) timer->stop();
    }
}
```

## IOperator Interface

### Definition

```cpp
// From execute/operator.h
class IOperator {
public:
    virtual ~IOperator() = default;

    // Get operator name
    virtual std::string get_operator_name() const = 0;

    // Evaluate operator
    virtual void Eval(IStorageInterface& storage,
                      ParamsMap& params,
                      Context&& ctx,
                      OprTimer* timer) = 0;
};
```

### Operator Categories

| Category | Operators | Directory |
|----------|-----------|-----------|
| **Retrieve** | Scan, Edge, Path, Project, Filter, Join, GroupBy, OrderBy, Limit | `ops/retrieve/` |
| **Insert** | CreateVertex, CreateEdge | `ops/insert/` |
| **Batch** | BatchInsert, BatchDelete, DataSource, DataExport | `ops/batch/` |
| **DDL** | CreateVertexType, CreateEdgeType, Drop | `ops/ddl/` |
| **Admin** | Checkpoint, Extension | `ops/admin/` |

## PlanParser

Converts Protocol Buffers to Pipeline.

```cpp
// From execute/plan_parser.h
class PlanParser {
public:
    static PlanParser& get() {
        static PlanParser instance;
        return instance;
    }

    // Parse physical plan
    Pipeline parse_execute_pipeline(
        const physical::PhysicalPlan& plan,
        const Schema& schema,
        ContextMeta& ctx_meta);

    // Parse parameter types
    std::unordered_map<std::string, DataType> parse_params_type(
        const physical::PhysicalPlan& plan);

private:
    // Operator builders
    std::unordered_map<std::string, std::unique_ptr<IOperatorBuilder>> builders_;

    void registerBuilders();
};
```

### Parsing Logic

```cpp
Pipeline PlanParser::parse_execute_pipeline(
    const physical::PhysicalPlan& plan,
    const Schema& schema,
    ContextMeta& ctx_meta) {

    Pipeline pipeline;

    for (int i = 0; i < plan.plan_size(); ++i) {
        const auto& physicalOpr = plan.plan(i);
        const auto& opr = physicalOpr.opr();

        // Get builder for operator type
        std::string opType = getOperatorType(opr);
        auto& builder = builders_.at(opType);

        // Build operator
        auto result = builder->Build(schema, ctx_meta, plan, i);

        // Add to pipeline
        pipeline.AddOperator(std::move(result.operator));

        // Update context metadata
        ctx_meta = std::move(result.ctx_meta);
    }

    return pipeline;
}
```

## Operator Implementations

### Scan Operator

```cpp
// From ops/retrieve/scan.cc
class ScanOperator : public IOperator {
public:
    ScanOperator(const std::vector<label_t>& labels,
                 std::unique_ptr<IPredicate> predicate,
                 int32_t alias);

    void Eval(IStorageInterface& storage,
              ParamsMap& params,
              Context&& ctx,
              OprTimer* timer) override;

private:
    std::vector<label_t> labels_;
    std::unique_ptr<IPredicate> predicate_;
    int32_t alias_;
};

void ScanOperator::Eval(IStorageInterface& storage,
                         ParamsMap& params,
                         Context&& ctx,
                         OprTimer* timer) {
    // Create vertex column
    auto column = std::make_shared<VertexColumn>();

    // Get vertices from storage
    for (auto label : labels_) {
        auto vertexSet = storage.GetVertexSet(label);
        for (vid_t vid : vertexSet) {
            if (predicate_ && !predicate_->eval(vid, storage)) {
                continue;
            }
            column->append(vid);
        }
    }

    // Update context
    ctx.set(alias_, std::move(column));
}
```

### EdgeExpand Operator

```cpp
// From ops/retrieve/edge.cc
class EdgeExpandOperator : public IOperator {
public:
    EdgeExpandOperator(uint32_t triplet_id,
                       Direction direction,
                       ExpandOpt opt,
                       int32_t start_alias,
                       int32_t alias);

    void Eval(IStorageInterface& storage,
              ParamsMap& params,
              Context&& ctx,
              OprTimer* timer) override;

private:
    uint32_t triplet_id_;
    Direction direction_;
    ExpandOpt opt_;
    int32_t start_alias_;
    int32_t alias_;
};

void EdgeExpandOperator::Eval(IStorageInterface& storage,
                                ParamsMap& params,
                                Context&& ctx,
                                OprTimer* timer) {
    auto& input_col = ctx.get<VertexColumn>(start_alias_);

    // Create output column
    auto output_col = std::make_shared<VertexColumn>();

    // Get edge view
    auto view = (direction_ == Direction::OUT)
        ? storage.GetOutgoingEdges(triplet_id_)
        : storage.GetIncomingEdges(triplet_id_);

    // Expand edges
    for (vid_t src : input_col) {
        for (auto& nbr : view.get_edges(src)) {
            output_col->append(nbr.neighbor());
        }
    }

    ctx.set(alias_, std::move(output_col));
}
```

### Filter Operator

```cpp
// From ops/retrieve/filter.cc
class FilterOperator : public IOperator {
public:
    FilterOperator(std::unique_ptr<ExpressionEvaluator> predicate);

    void Eval(IStorageInterface& storage,
              ParamsMap& params,
              Context&& ctx,
              OprTimer* timer) override;

private:
    std::unique_ptr<ExpressionEvaluator> predicate_;
};

void FilterOperator::Eval(IStorageInterface& storage,
                           ParamsMap& params,
                           Context&& ctx,
                           OprTimer* timer) {
    // Evaluate predicate for each row
    auto& selection = predicate_->eval(ctx, storage);

    // Filter all columns
    for (auto& [alias, column] : ctx.columns()) {
        column->filter(selection);
    }
}
```

### Project Operator

```cpp
// From ops/retrieve/project.cc
class ProjectOperator : public IOperator {
public:
    ProjectOperator(std::vector<std::pair<int32_t, std::unique_ptr<ExpressionEvaluator>>> mappings);

    void Eval(IStorageInterface& storage,
              ParamsMap& params,
              Context&& ctx,
              OprTimer* timer) override;

private:
    std::vector<std::pair<int32_t, std::unique_ptr<ExpressionEvaluator>>> mappings_;
};

void ProjectOperator::Eval(IStorageInterface& storage,
                             ParamsMap& params,
                             Context&& ctx,
                             OprTimer* timer) {
    // Create new context with projected columns
    Context newCtx;

    for (auto& [alias, expr] : mappings_) {
        auto column = expr->eval(ctx, storage);
        newCtx.set(alias, std::move(column));
    }

    ctx = std::move(newCtx);
}
```

## Context

### Definition

```cpp
// From common/context.h
class Context {
public:
    Context() = default;

    // Column access
    template <typename T>
    std::shared_ptr<T> get(int32_t alias) const {
        return std::dynamic_pointer_cast<T>(columns_.at(alias));
    }

    void set(int32_t alias, std::shared_ptr<IContextColumn> column) {
        columns_[alias] = std::move(column);
    }

    void remove(int32_t alias) {
        columns_.erase(alias);
    }

    // Row count
    size_t row_num() const {
        if (columns_.empty()) return 0;
        return columns_.begin()->second->size();
    }

    // Merge contexts (for UNION)
    void union_ctx(Context&& other);

    // Get all columns
    const std::unordered_map<int32_t, std::shared_ptr<IContextColumn>>& columns() const {
        return columns_;
    }

private:
    std::unordered_map<int32_t, std::shared_ptr<IContextColumn>> columns_;
    std::vector<int32_t> tag_ids_;
};
```

## Performance Considerations

### Operator Fusion

Some operators can be fused for better performance:

```cpp
// Before: Separate operators
Scan → EdgeExpand → GetV → Filter

// After: Fused operator
ScanFilteredEdge  // Combines all four
```

### Vectorized Execution

Context columns enable vectorized operations:

```cpp
// Instead of row-by-row
for (vid_t vid : vertices) {
    if (predicate(vid)) {
        output.append(vid);
    }
}

// Vectorized
auto mask = predicate_vectorized(vertices);
output.append_filtered(vertices, mask);
```

### Memory Management

Columns use reference counting:

```cpp
// Column is shared, not copied
auto column = ctx.get<VertexColumn>(alias);
auto filtered = column->filter(mask);  // Creates view, not copy
```

## References

- [Context Columns](./context-columns.md)
- [Operator Implementations](./operator-implementations.md)
- [Plan Parser](./plan-parser.md)