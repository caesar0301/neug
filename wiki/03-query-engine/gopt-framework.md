# GOpt Framework

This document describes NeuG's graph optimization framework.

## Overview

GOpt is NeuG's unified query optimization framework, adapted from GraphScope:

- **Protocol Buffers IR**: Physical plans serialized as PB
- **Graph-specific optimization**: Optimized for graph patterns
- **GQL-ready**: Unified IR design for future GQL support

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      GOpt Framework                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   LogicalPlan                                                   │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    GOptPlanner                           │  │
│   │                                                         │  │
│   │   compilePlan(query) → (PhysicalPlan, query_string)    │  │
│   └─────────────────────────────────────────────────────────┘  │
│       │                                                         │
│       ├──────────────────┬──────────────────┐                  │
│       ▼                  ▼                  ▼                  │
│   ┌─────────┐      ┌───────────┐      ┌───────────┐           │
│   │GCatalog │      │GAliasMgr  │      │GTypeConv  │           │
│   │(Schema) │      │(Aliases)  │      │ (Types)   │           │
│   └─────────┘      └───────────┘      └───────────┘           │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                 GQueryConvertor                          │  │
│   │                                                         │  │
│   │   convert(LogicalPlan) → PhysicalPlan (Protocol Buffers)│  │
│   └─────────────────────────────────────────────────────────┘  │
│       │                                                         │
│       ├──────────────────┬──────────────────┐                  │
│       ▼                  ▼                  ▼                  │
│   ┌─────────┐      ┌───────────┐      ┌───────────┐           │
│   │GExprConv│      │GDDLConv   │      │GTypeConv  │           │
│   │(Exprs)  │      │  (DDL)    │      │ (Types)   │           │
│   └─────────┘      └───────────┘      └───────────┘           │
│       │                                                         │
│       ▼                                                         │
│   PhysicalPlan (Protocol Buffers)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## GOptPlanner

Main entry point for query compilation.

```cpp
// From gopt_planner.h
class GOptPlanner : public IGraphPlanner {
public:
    GOptPlanner() {
        database = std::make_unique<MetadataManager>();
        ctx = std::make_unique<ClientContext>(database.get());
        MetadataRegistry::registerMetadata(database.get());
    }

    result<std::pair<PhysicalPlan, std::string>> compilePlan(
        const std::string& query) override;

    void update_meta(const YAML::Node& schema) override;
    void update_statistics(const std::string& stats_json) override;

private:
    std::unique_ptr<MetadataManager> database;
    std::unique_ptr<ClientContext> ctx;
};
```

### compilePlan Implementation

```cpp
result<std::pair<PhysicalPlan, std::string>> GOptPlanner::compilePlan(
    const std::string& query) {

    // 1. Parse query
    auto parsed = Parser::parseQuery(query);

    // 2. Bind to schema
    Binder binder(ctx.get());
    auto bound = binder.bind(parsed.get());

    // 3. Generate logical plan
    Planner planner(ctx.get());
    auto logicalPlan = planner.plan(bound.get());

    // 4. Optimize logical plan
    Optimizer optimizer(ctx.get());
    optimizer.optimize(&logicalPlan);

    // 5. Convert to physical plan
    auto aliasMgr = std::make_shared<GAliasManager>(logicalPlan);
    GQueryConvertor convertor(aliasMgr, ctx->getCatalog());
    auto physicalPlan = convertor.convert(logicalPlan, false);

    // 6. Serialize to string
    std::string planStr;
    physicalPlan->SerializeToString(&planStr);

    return std::make_pair(std::move(*physicalPlan), planStr);
}
```

## GCatalog

Schema management for GOpt.

```cpp
// From g_catalog.h
class GCatalog : public Catalog {
public:
    GCatalog();
    GCatalog(const std::filesystem::path& schemaPath);
    GCatalog(const std::string& schemaData);
    GCatalog(const YAML::Node& schema);

    void updateSchema(const YAML::Node& schema);

    // Function management
    void addFunctionWithSignature(const std::string& name,
                                   function_set functions);
    Function* getFunctionWithSignature(const std::string& signature);

private:
    void loadSchema(const YAML::Node& schema);
    void registerBuiltInFunctions();

    std::unique_ptr<NodeTableCatalogEntry> createNodeTableEntry(
        const YAML::Node& info);
    std::unique_ptr<GRelTableCatalogEntry> createRelTableEntry(
        const YAML::Node& info);
};
```

### Schema Loading

```cpp
void GCatalog::loadSchema(const YAML::Node& schema) {
    auto info = schema["schema"];

    // Load vertex types
    for (const auto& vertex : info["vertex_types"]) {
        auto entry = createNodeTableEntry(vertex);
        tables_->emplace(std::move(entry));
    }

    // Load edge types
    for (const auto& edge : info["edge_types"]) {
        auto labelName = edge["type_name"].as<std::string>();
        auto relations = edge["vertex_type_pair_relations"];

        for (const auto& rel : relations) {
            auto entry = createRelTableEntry(rel, labelName);
            tables_->emplace(std::move(entry));
        }
    }
}
```

## GAliasManager

Maps query aliases to integer IDs.

```cpp
// From g_alias_manager.h
class GAliasManager {
public:
    GAliasManager(const LogicalPlan& plan);

    alias_id_t getAliasId(const std::string& name);
    GAliasName getGAliasName(alias_id_t id);

    static void extractGAliasNames(const LogicalOperator& op,
                                   std::vector<GAliasName>& names);

private:
    std::unordered_map<std::string, alias_id_t> uniqueNameToId;
    std::unordered_map<alias_id_t, GAliasName> idToGName;
    alias_id_t nextId{0};
};

// Special IDs
constexpr alias_id_t DEFAULT_ALIAS_ID = -1;
const std::string DEFAULT_ALIAS_NAME = "__default_alias_name__";
```

### Alias Extraction

```cpp
void GAliasManager::extractGAliasNames(const LogicalOperator& op,
                                        std::vector<GAliasName>& names) {
    switch (op.getOperatorType()) {
        case LogicalOperatorType::SCAN_NODE_TABLE: {
            auto scan = op.constPtrCast<LogicalScanNodeTable>();
            names.push_back(GAliasName(scan->getAliasName(), scan->getAliasName()));
            break;
        }
        case LogicalOperatorType::EXTEND: {
            auto extend = op.constPtrCast<LogicalExtend>();
            names.push_back(GAliasName(extend->getAliasName(), extend->getAliasName()));
            break;
        }
        case LogicalOperatorType::PROJECTION: {
            auto proj = op.constPtrCast<LogicalProjection>();
            for (auto& expr : proj->getExpressions()) {
                names.push_back(GAliasName(expr->getUniqueName(), expr->getUniqueName()));
            }
            break;
        }
        // ...
    }

    // Recurse into children
    for (auto& child : op.getChildren()) {
        extractGAliasNames(*child, names);
    }
}
```

## GQueryConvertor

Converts logical plan to physical plan (Protocol Buffers).

```cpp
// From g_query_converter.h
class GQueryConvertor {
public:
    GQueryConvertor(std::shared_ptr<GAliasManager> aliasMgr, Catalog* catalog);

    std::unique_ptr<physical::PhysicalPlan> convert(
        const LogicalPlan& plan, bool skipSink);

private:
    void convertOperator(const LogicalOperator& op, PhysicalPlan* plan);
    void convertScan(const LogicalScanNodeTable& scan, PhysicalPlan* plan);
    void convertExtend(const LogicalExtend& extend, PhysicalPlan* plan);
    void convertFilter(const LogicalFilter& filter, PhysicalPlan* plan);
    void convertProjection(const LogicalProjection& proj, PhysicalPlan* plan);
    void convertAggregate(const LogicalAggregate& agg, PhysicalPlan* plan);
    // ... more operators

    std::shared_ptr<GAliasManager> aliasManager;
    std::unique_ptr<GExprConverter> exprConvertor;
    std::unique_ptr<GPhysicalTypeConverter> typeConverter;
    Catalog* catalog;
};
```

### Conversion Process

```cpp
std::unique_ptr<physical::PhysicalPlan> GQueryConvertor::convert(
    const LogicalPlan& plan, bool skipSink) {

    auto planPB = std::make_unique<physical::PhysicalPlan>();

    // Traverse logical plan tree
    convertOperator(*plan.getLastOperator(), planPB.get());

    // Add sink operator
    if (!skipSink) {
        auto sink = std::make_unique<physical::Sink>();
        auto physicalOpr = std::make_unique<physical::PhysicalOpr>();
        auto opr = std::make_unique<physical::PhysicalOpr_Operator>();
        opr->set_allocated_sink(sink.release());
        physicalOpr->set_allocated_opr(opr.release());
        planPB->mutable_plan()->AddAllocated(physicalOpr.release());
    }

    return planPB;
}

void GQueryConvertor::convertOperator(const LogicalOperator& op,
                                       PhysicalPlan* plan) {
    // Process children first (bottom-up)
    for (auto& child : op.getChildren()) {
        convertOperator(*child, plan);
    }

    // Convert this operator
    switch (op.getOperatorType()) {
        case LogicalOperatorType::SCAN_NODE_TABLE:
            convertScan(op.constPtrCast<LogicalScanNodeTable>(), plan);
            break;
        case LogicalOperatorType::EXTEND:
            convertExtend(op.constPtrCast<LogicalExtend>(), plan);
            break;
        case LogicalOperatorType::FILTER:
            convertFilter(op.constPtrCast<LogicalFilter>(), plan);
            break;
        // ...
    }
}
```

### Scan Conversion

```cpp
void GQueryConvertor::convertScan(const LogicalScanNodeTable& scan,
                                   PhysicalPlan* plan) {
    auto scanPB = std::make_unique<physical::Scan>();
    scanPB->set_scan_opt(physical::Scan_ScanOpt_VERTEX);

    // Set table IDs
    auto params = std::make_unique<algebra::QueryParams>();
    for (auto& labelId : scan.getTableIDs()) {
        auto tableId = std::make_unique<common::NameOrId>();
        tableId->set_id(labelId);
        params->mutable_tables()->AddAllocated(tableId.release());
    }
    scanPB->set_allocated_params(params.release());

    // Set alias
    auto aliasId = aliasManager->getAliasId(scan.getAliasName());
    if (aliasId != DEFAULT_ALIAS_ID) {
        auto aliasValue = std::make_unique<google::protobuf::Int32Value>();
        aliasValue->set_value(aliasId);
        scanPB->set_allocated_alias(aliasValue.release());
    }

    // Set metadata (type info)
    auto metaData = std::make_unique<physical::PhysicalOpr_MetaData>();
    auto nodeType = scan.getNodeType(catalog);
    metaData->set_allocated_type(typeConverter->convertNodeType(*nodeType).release());
    metaData->set_alias(aliasId);

    // Build physical operator
    auto physicalPB = std::make_unique<physical::PhysicalOpr>();
    auto oprPB = std::make_unique<physical::PhysicalOpr_Operator>();
    oprPB->set_allocated_scan(scanPB.release());
    physicalPB->set_allocated_opr(oprPB.release());
    physicalPB->mutable_meta_data()->AddAllocated(metaData.release());

    plan->mutable_plan()->AddAllocated(physicalPB.release());
}
```

## GExprConverter

Converts binder expressions to Protocol Buffers.

```cpp
// From g_expr_converter.h
class GExprConverter {
public:
    GExprConverter(std::shared_ptr<GAliasManager> aliasMgr);

    std::unique_ptr<common::Expression> convert(
        const Expression& expr, const LogicalOperator& child);

    std::unique_ptr<physical::GroupBy_AggFunc> convertAggFunc(
        const AggregateFunctionExpression& expr, const LogicalOperator& child);

private:
    std::unique_ptr<common::Expression> convertLiteral(const LiteralExpression& expr);
    std::unique_ptr<common::Expression> convertProperty(const PropertyExpression& expr);
    std::unique_ptr<common::Expression> convertVariable(const VariableExpression& expr);
    std::unique_ptr<common::Expression> convertFunction(const ScalarFunctionExpression& expr);
    std::unique_ptr<common::Expression> convertPattern(const NodeOrRelExpression& expr);

    std::shared_ptr<GAliasManager> aliasManager;
    GLogicalTypeConverter typeConverter;
};
```

### Expression Conversion

```cpp
std::unique_ptr<common::Expression> GExprConverter::convert(
    const Expression& expr, const LogicalOperator& child) {

    switch (expr.expressionType) {
        case ExpressionType::LITERAL:
            return convertLiteral(static_cast<const LiteralExpression&>(expr));

        case ExpressionType::PROPERTY:
            return convertProperty(static_cast<const PropertyExpression&>(expr));

        case ExpressionType::VARIABLE:
            return convertVariable(static_cast<const VariableExpression&>(expr));

        case ExpressionType::FUNCTION:
            return convertFunction(static_cast<const ScalarFunctionExpression&>(expr));

        case ExpressionType::PATTERN:
            return convertPattern(static_cast<const NodeOrRelExpression&>(expr));

        case ExpressionType::EQUALS:
        case ExpressionType::NOT_EQUALS:
        case ExpressionType::GREATER_THAN:
        case ExpressionType::LESS_THAN:
        case ExpressionType::AND:
        case ExpressionType::OR:
            return convertBinaryOp(expr, child);

        // ...
    }
}
```

## GPhysicalTypeConverter

Converts logical types to Protocol Buffers types.

```cpp
// From g_type_converter.h
class GPhysicalTypeConverter {
public:
    std::unique_ptr<common::IrDataType> convertNodeType(const GNodeType& nodeType);
    std::unique_ptr<common::IrDataType> convertRelType(const GRelType& relType);
    std::unique_ptr<common::IrDataType> convertPathType(const GRelType& relType);
    std::unique_ptr<common::IrDataType> convertLogicalType(const LogicalType& type);

private:
    std::unique_ptr<common::GraphDataType::GraphElementType> convertNodeTable(
        NodeTableCatalogEntry* entry);
    std::unique_ptr<common::GraphDataType::GraphElementType> convertRelTable(
        GRelTableCatalogEntry* entry);
};
```

## Protocol Buffer Definitions

### PhysicalPlan

```protobuf
// From proto/plan/physical.proto
message PhysicalPlan {
    repeated PhysicalOpr plan = 1;
}

message PhysicalOpr {
    PhysicalOpr_Operator opr = 1;
    repeated MetaData meta_data = 2;
}

message PhysicalOpr_Operator {
    oneof opr {
        Scan scan = 1;
        EdgeExpand edge = 2;
        PathExpand path = 3;
        GetV vertex = 4;
        Project project = 5;
        GroupBy group_by = 6;
        Join join = 7;
        Sink sink = 8;
        // ...
    }
}

message MetaData {
    int32 alias = 1;
    common.IrDataType type = 2;
}
```

### Graph Operators

```protobuf
message Scan {
    ScanOpt scan_opt = 1;  // VERTEX or EDGE
    algebra.QueryParams params = 2;
    google.protobuf.Int32Value alias = 3;
    algebra.IndexPredicate idx_predicate = 4;
}

message EdgeExpand {
    ExpandOpt expand_opt = 1;  // VERTEX, EDGE, DEGREE
    Direction direction = 2;   // OUT, IN, BOTH
    algebra.QueryParams params = 3;
    google.protobuf.Int32Value alias = 4;
    google.protobuf.Int32Value v_tag = 5;
    bool is_optional = 6;
}

message PathExpand {
    ExpandBase base = 1;
    algebra.Range hop_range = 2;
    PathOpt path_opt = 3;
    ResultOpt result_opt = 4;
    google.protobuf.Int32Value alias = 5;
}
```

## References

- [Optimizer Rules](./optimizer-rules.md)
- [Physical Operators](./physical-operators.md)
- [Expression Evaluation](./expression-evaluation.md)