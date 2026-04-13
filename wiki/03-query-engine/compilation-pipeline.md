# Compilation Pipeline

This document traces NeuG's query compilation pipeline from Cypher to logical plan.

## Pipeline Overview

```
Cypher Query String
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                     PARSING STAGE                              │
│                                                               │
│   ┌─────────────┐    ┌──────────────┐    ┌───────────────┐  │
│   │   Lexer     │───▶│    Parser    │───▶│   Transform   │  │
│   │  (ANTLR4)   │    │  (Cypher.g4) │    │   (To AST)    │  │
│   └─────────────┘    └──────────────┘    └───────────────┘  │
│                                                   │           │
│                                                   ▼           │
│                                          ParsedExpression     │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                     BINDING STAGE                              │
│                                                               │
│   ┌─────────────┐    ┌──────────────┐    ┌───────────────┐  │
│   │  Catalog    │    │   Binder     │    │   Expression  │  │
│   │  Lookup     │───▶│  Resolution  │───▶│   Binding     │  │
│   └─────────────┘    └──────────────┘    └───────────────┘  │
│                                                   │           │
│                                                   ▼           │
│                                           BoundStatement      │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                     PLANNING STAGE                             │
│                                                               │
│   ┌─────────────┐    ┌──────────────┐    ┌───────────────┐  │
│   │  Statement  │    │   Query      │    │    Logical    │  │
│   │  Planner    │───▶│   Planner    │───▶│   Operators   │  │
│   └─────────────┘    └──────────────┘    └───────────────┘  │
│                                                   │           │
│                                                   ▼           │
│                                            LogicalPlan        │
└───────────────────────────────────────────────────────────────┘
```

## Stage 1: Parsing

### ANTLR4 Lexer

Tokenizes the Cypher query:

```cpp
// From antlr_parser/
// CypherLexer.g4 grammar
MATCH    : 'MATCH';
WHERE    : 'WHERE';
RETURN   : 'RETURN';
CREATE   : 'CREATE';
DELETE   : 'DELETE';
SET      : 'SET';
IDENTIFIER : [a-zA-Z_][a-zA-Z0-9_]*;
STRING    : '\'' (~'\'' | '\'\'')* '\'';
NUMBER    : [0-9]+ ('.' [0-9]+)?;
```

### ANTLR4 Parser

Generates parse tree:

```cpp
// From CypherParser.g4
query
    : statement ( ';' )? EOF
    ;

statement
    : queryStatement
    | updateStatement
    | ddlStatement
    ;

queryStatement
    : MATCH matchPattern (WHERE expression)? RETURN returnItems
    ;

matchPattern
    : nodePattern (relationshipPattern nodePattern)*
    ;
```

### Transform to ParsedExpression

```cpp
// From transform/
class ParseTreeTransformer {
public:
    std::unique_ptr<ParsedExpression> transform(CypherParser::QueryContext* ctx) {
        auto statement = transformStatement(ctx->statement());
        return statement;
    }

private:
    std::unique_ptr<ParsedStatement> transformStatement(CypherParser::StatementContext* ctx) {
        if (ctx->queryStatement()) {
            return transformQueryStatement(ctx->queryStatement());
        } else if (ctx->updateStatement()) {
            return transformUpdateStatement(ctx->updateStatement());
        }
        // ...
    }

    std::unique_ptr<ParsedQueryStatement> transformQueryStatement(
        CypherParser::QueryStatementContext* ctx) {

        auto matchPattern = transformMatchPattern(ctx->matchPattern());
        auto where = ctx->WHERE() ? transformExpression(ctx->expression()) : nullptr;
        auto returnItems = transformReturnItems(ctx->returnItems());

        auto stmt = std::make_unique<ParsedQueryStatement>();
        stmt->setMatchPattern(std::move(matchPattern));
        stmt->setWhere(std::move(where));
        stmt->setReturnItems(std::move(returnItems));
        return stmt;
    }
};
```

## Stage 2: Binding

### Binder Overview

```cpp
// From binder/
class Binder {
    Catalog* catalog_;
    std::unordered_map<std::string, std::shared_ptr<Expression>> variables_;

public:
    std::unique_ptr<BoundStatement> bind(const ParsedStatement* stmt) {
        switch (stmt->getType()) {
            case StatementType::QUERY:
                return bindQueryStatement(static_cast<const ParsedQueryStatement*>(stmt));
            case StatementType::UPDATE:
                return bindUpdateStatement(static_cast<const ParsedUpdateStatement*>(stmt));
            // ...
        }
    }

private:
    std::unique_ptr<BoundQueryStatement> bindQueryStatement(
        const ParsedQueryStatement* stmt) {

        // 1. Bind match pattern
        auto boundPattern = bindMatchPattern(stmt->getMatchPattern());

        // 2. Bind WHERE clause
        auto boundWhere = stmt->getWhere() ? bindExpression(stmt->getWhere()) : nullptr;

        // 3. Bind RETURN clause
        auto boundReturn = bindReturnItems(stmt->getReturnItems());

        auto result = std::make_unique<BoundQueryStatement>();
        result->setPattern(std::move(boundPattern));
        result->setWhere(std::move(boundWhere));
        result->setReturnItems(std::move(boundReturn));
        return result;
    }
};
```

### Node Binding

```cpp
std::shared_ptr<NodeExpression> Binder::bindNodePattern(
    const ParsedNodePattern* pattern) {

    // 1. Get or create variable
    std::string varName = pattern->getVariable();
    if (variables_.count(varName)) {
        throw BinderException("Variable already defined: " + varName);
    }

    // 2. Resolve labels
    std::vector<NodeTableCatalogEntry*> entries;
    for (const auto& label : pattern->getLabels()) {
        auto* entry = catalog_->getTableCatalogEntry(label);
        if (!entry) {
            throw BinderException("Unknown label: " + label);
        }
        entries.push_back(entry->ptrCast<NodeTableCatalogEntry>());
    }

    // 3. Create node expression
    auto nodeExpr = std::make_shared<NodeExpression>(varName, entries);
    variables_[varName] = nodeExpr;

    // 4. Bind properties
    for (const auto& prop : pattern->getProperties()) {
        auto propExpr = bindPropertyExpression(prop, nodeExpr);
        nodeExpr->addProperty(propExpr);
    }

    return nodeExpr;
}
```

### Relationship Binding

```cpp
std::shared_ptr<RelExpression> Binder::bindRelPattern(
    const ParsedRelPattern* pattern,
    std::shared_ptr<NodeExpression> src,
    std::shared_ptr<NodeExpression> dst) {

    // 1. Get or create variable
    std::string varName = pattern->getVariable();

    // 2. Resolve labels
    std::vector<GRelTableCatalogEntry*> entries;
    for (const auto& label : pattern->getLabels()) {
        // Find edges connecting src type to dst type
        auto relEntries = catalog_->getRelTables(src->getLabelIds(),
                                                   dst->getLabelIds(),
                                                   label);
        entries.insert(entries.end(), relEntries.begin(), relEntries.end());
    }

    // 3. Create relationship expression
    auto relExpr = std::make_shared<RelExpression>(varName, entries, src, dst);

    // 4. Set direction
    relExpr->setDirection(pattern->getDirection());

    // 5. Set variable length (if specified)
    if (pattern->hasVariableLength()) {
        relExpr->setVariableLength(pattern->getLowerBound(), pattern->getUpperBound());
    }

    return relExpr;
}
```

### Expression Binding

```cpp
std::shared_ptr<Expression> Binder::bindExpression(
    const ParsedExpression* expr) {

    switch (expr->getType()) {
        case ExpressionType::PROPERTY: {
            auto propExpr = static_cast<const ParsedPropertyExpression*>(expr);
            auto var = variables_.at(propExpr->getVariableName());
            return bindPropertyAccess(var, propExpr->getPropertyName());
        }
        case ExpressionType::FUNCTION: {
            auto funcExpr = static_cast<const ParsedFunctionExpression*>(expr);
            return bindFunctionCall(funcExpr);
        }
        case ExpressionType::LITERAL: {
            auto litExpr = static_cast<const ParsedLiteralExpression*>(expr);
            return std::make_shared<LiteralExpression>(litExpr->getValue());
        }
        case ExpressionType::BINARY_OP: {
            auto binExpr = static_cast<const ParsedBinaryOpExpression*>(expr);
            auto left = bindExpression(binExpr->getLeft());
            auto right = bindExpression(binExpr->getRight());
            return std::make_shared<BinaryOpExpression>(binExpr->getOp(), left, right);
        }
        // ...
    }
}
```

## Stage 3: Planning

### Query Planner

```cpp
// From planner/
class QueryPlanner {
    Catalog* catalog_;

public:
    std::unique_ptr<LogicalPlan> plan(const BoundQueryStatement* stmt) {
        auto plan = std::make_unique<LogicalPlan>();

        // 1. Plan MATCH pattern
        auto lastOp = planPattern(stmt->getPattern());
        plan->setLastOperator(lastOp);

        // 2. Plan WHERE clause
        if (stmt->getWhere()) {
            lastOp = planFilter(stmt->getWhere(), lastOp);
            plan->setLastOperator(lastOp);
        }

        // 3. Plan RETURN clause
        lastOp = planProjection(stmt->getReturnItems(), lastOp);
        plan->setLastOperator(lastOp);

        return plan;
    }

private:
    std::shared_ptr<LogicalOperator> planPattern(const BoundMatchPattern& pattern) {
        std::shared_ptr<LogicalOperator> lastOp;

        for (const auto& element : pattern.getElements()) {
            if (element->isNode()) {
                auto node = element->asNode();
                if (lastOp == nullptr) {
                    // First node - scan
                    lastOp = std::make_shared<LogicalScanNodeTable>(node);
                } else {
                    // Node connected by relationship - get vertex
                    lastOp = std::make_shared<LogicalGetV>(node, lastOp);
                }
            } else {
                // Relationship - extend
                auto rel = element->asRel();
                auto src = rel->getSrcNode();
                auto dst = rel->getDstNode();

                if (rel->hasVariableLength()) {
                    lastOp = std::make_shared<LogicalRecursiveExtend>(rel, src, dst, lastOp);
                } else {
                    lastOp = std::make_shared<LogicalExtend>(rel, src, dst, lastOp);
                }
            }
        }

        return lastOp;
    }
};
```

### Logical Operator Types

```cpp
// From planner/operator/
enum class LogicalOperatorType {
    SCAN_NODE_TABLE,
    EXTEND,
    RECURSIVE_EXTEND,
    GET_V,
    FILTER,
    PROJECTION,
    AGGREGATE,
    DISTINCT,
    ORDER_BY,
    LIMIT,
    HASH_JOIN,
    CROSS_PRODUCT,
    INTERSECT,
    UNION_ALL,
    INSERT,
    DELETE,
    SET_PROPERTY,
    COPY_FROM,
    COPY_TO,
    // ...
};
```

### Example: Planning a Query

**Query**:
```cypher
MATCH (n:person)-[r:knows]->(m:person)
WHERE n.age > 30
RETURN n.name, m.name
```

**Generated Logical Plan**:
```
LogicalProjection [n.name, m.name]
└── LogicalFilter (n.age > 30)
    └── LogicalGetV (m)
        └── LogicalExtend [r:knows] (n → m)
            └── LogicalScanNodeTable [n:person]
```

## Stage 4: Optimization

See [optimizer-rules.md](./optimizer-rules.md) for detailed optimization pipeline.

```cpp
// From optimizer/
void Optimizer::optimize(LogicalPlan* plan) {
    // Apply optimization rules in sequence
    RemoveFactorizationRewriter().rewrite(plan);
    FilterPushDownOptimizer().rewrite(plan);
    ProjectionPushDownOptimizer().rewrite(plan);
    ExpandGetVFusion().rewrite(plan);
    // ... more rules
}
```

## Statement Types

### Query Statements

| Type | Cypher | Description |
|------|--------|-------------|
| MATCH | `MATCH ... RETURN ...` | Read query |
| OPTIONAL MATCH | `OPTIONAL MATCH ...` | Optional pattern |

### Update Statements

| Type | Cypher | Description |
|------|--------|-------------|
| CREATE | `CREATE (n:Label)` | Create vertices/edges |
| DELETE | `DELETE n` | Remove vertices/edges |
| SET | `SET n.prop = val` | Update properties |
| MERGE | `MERGE (n:Label)` | Create if not exists |

### DDL Statements

| Type | Cypher | Description |
|------|--------|-------------|
| CREATE VERTEX TYPE | `CREATE VERTEX TYPE ...` | Define vertex type |
| CREATE EDGE TYPE | `CREATE EDGE TYPE ...` | Define edge type |
| DROP | `DROP VERTEX TYPE ...` | Remove type |

### Data Statements

| Type | Cypher | Description |
|------|--------|-------------|
| COPY FROM | `COPY Label FROM 'file'` | Bulk load |
| COPY TO | `COPY ... TO 'file'` | Export data |

## References

- [GOpt Framework](./gopt-framework.md)
- [Optimizer Rules](./optimizer-rules.md)
- [Logical Operators](./logical-operators.md)