# Schema Management

This document describes NeuG's schema definition and management system.

## Overview

NeuG's schema system defines:

- **Vertex types**: Labels, properties, primary keys
- **Edge types**: Labels, source/destination vertices, properties, multiplicity

Schema is defined in YAML format and managed by the `Schema` class.

## YAML Schema Format

### Complete Example

```yaml
schema:
  vertex_types:
    - type_name: person
      type_id: 0
      primary_keys: [id]
      properties:
        - property_name: id
          property_type: INT64
        - property_name: fName
          property_type: STRING
        - property_name: lName
          property_type: STRING
        - property_name: age
          property_type: INT32
        - property_name: created
          property_type: TIMESTAMP

    - type_name: company
      type_id: 1
      primary_keys: [name]
      properties:
        - property_name: name
          property_type: STRING
        - property_name: revenue
          property_type: DOUBLE
        - property_name: founded
          property_type: DATE

  edge_types:
    - type_name: knows
      type_id: 0
      vertex_type_pair_relations:
        - source_vertex: person
          destination_vertex: person
          relation: many_to_many
      properties:
        - property_name: since
          property_type: DATE
        - property_name: weight
          property_type: DOUBLE

    - type_name: works_at
      type_id: 1
      vertex_type_pair_relations:
        - source_vertex: person
          destination_vertex: company
          relation: many_to_one
      properties:
        - property_name: role
          property_type: STRING
        - property_name: start_date
          property_type: DATE
        - property_name: salary
          property_type: DOUBLE
```

## Vertex Type Definition

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type_name` | Yes | Vertex label name |
| `type_id` | Yes | Unique integer identifier |
| `primary_keys` | No | List of primary key property names (default: none) |
| `properties` | No | List of property definitions |
| `max_num` | No | Maximum number of vertices (optional) |

### Property Definition

| Field | Required | Description |
|-------|----------|-------------|
| `property_name` | Yes | Property name |
| `property_type` | Yes | Data type |

### Supported Types

| Type | Description | Example |
|------|-------------|---------|
| `BOOL` | Boolean | `true`, `false` |
| `INT32` | 32-bit integer | `-2147483648` to `2147483647` |
| `INT64` | 64-bit integer | Large integers |
| `FLOAT` | 32-bit float | `3.14` |
| `DOUBLE` | 64-bit float | `3.14159265359` |
| `STRING` | UTF-8 string | `"hello"` |
| `DATE` | Date (YYYY-MM-DD) | `date("2024-01-15")` |
| `TIMESTAMP` | Date-time | `timestamp("2024-01-15T10:30:00")` |
| `INTERVAL` | Time duration | `interval("P1Y2M3D")` |
| `LIST<T>` | List of type T | `[1, 2, 3]` |
| `STRUCT` | Nested structure | `{name: "Alice", age: 30}` |

## Edge Type Definition

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type_name` | Yes | Edge label name |
| `type_id` | Yes | Unique integer identifier |
| `vertex_type_pair_relations` | Yes | List of source-destination pairs |
| `properties` | No | List of property definitions |

### Vertex Type Pair Relation

| Field | Required | Description |
|-------|----------|-------------|
| `source_vertex` | Yes | Source vertex type name |
| `destination_vertex` | Yes | Destination vertex type name |
| `relation` | Yes | Edge multiplicity |

### Edge Multiplicity

| Relation | Description | Example |
|----------|-------------|---------|
| `one_to_one` | At most one edge per vertex pair | `married_to` |
| `one_to_many` | One source, multiple destinations | `has_child` |
| `many_to_one` | Multiple sources, one destination | `works_at` |
| `many_to_many` | No restriction | `knows`, `friend_of` |

## Schema Class

### Definition

```cpp
// From schema.h
class Schema {
    // Label name to ID mapping
    IdIndexer<std::string, label_t> vlabel_indexer_;
    IdIndexer<std::string, label_t> elabel_indexer_;

    // Schema definitions
    std::vector<std::shared_ptr<VertexSchema>> v_schemas_;
    std::unordered_map<uint32_t, std::shared_ptr<EdgeSchema>> e_schemas_;

    // Tombstone markers for deleted types
    Bitset vlabel_tomb_;
    Bitset elabel_tomb_;
    Bitset elabel_triplet_tomb_;

public:
    // Load/Dump
    void LoadFromYaml(const YAML::Node& node);
    void DumpToYaml(const std::string& path);

    // Label operations
    label_t get_vertex_label_id(const std::string& name) const;
    label_t get_edge_label_id(const std::string& name) const;
    const std::string& get_vertex_label_name(label_t id) const;
    const std::string& get_edge_label_name(label_t id) const;

    // Schema access
    VertexSchema* get_vertex_schema(label_t label);
    EdgeSchema* get_edge_schema(uint32_t triplet_id);

    // Edge triplet ID
    uint32_t generate_edge_label(label_t src, label_t dst, label_t edge);
};
```

### VertexSchema

```cpp
// From schema.h
struct VertexSchema {
    std::string label_name;
    label_t label_id;

    std::vector<std::string> primary_keys;
    std::vector<DataType> property_types;
    std::vector<std::string> property_names;
    std::vector<std::string> default_property_values;

    size_t max_num;

    // Soft deletion
    std::vector<bool> vprop_soft_deleted;

    // Property access
    int getPropertyIndex(const std::string& name) const;
    const DataType& getPropertyType(int idx) const;
};
```

### EdgeSchema

```cpp
// From schema.h
struct EdgeSchema {
    std::string src_label_name;
    std::string dst_label_name;
    std::string edge_label_name;

    label_t src_label_id;
    label_t dst_label_id;
    label_t edge_label_id;

    EdgeStrategy oe_strategy;  // Outgoing: Multiple/Single/None
    EdgeStrategy ie_strategy;  // Incoming: Multiple/Single/None

    bool oe_mutable;
    bool ie_mutable;

    std::vector<DataType> property_types;
    std::vector<std::string> property_names;

    // Soft deletion
    std::vector<bool> eprop_soft_deleted;
};
```

## Schema Operations

### Loading Schema

```cpp
// From YAML file
Schema schema;
schema.LoadFromYaml(YAML::LoadFile("schema.yaml"));

// From YAML string
schema.LoadFromYaml(YAML::Load(yaml_string));
```

### Creating Vertex Type

```cpp
// DDL: CREATE VERTEX TYPE
void Schema::CreateVertexType(const std::string& name,
                               const std::vector<PropertyDef>& properties,
                               const std::vector<std::string>& primary_keys) {
    label_t id = vlabel_indexer_.insert(name);

    auto vs = std::make_shared<VertexSchema>();
    vs->label_name = name;
    vs->label_id = id;
    vs->primary_keys = primary_keys;
    // ... set properties

    v_schemas_.resize(id + 1);
    v_schemas_[id] = std::move(vs);
}
```

### Creating Edge Type

```cpp
// DDL: CREATE EDGE TYPE
void Schema::CreateEdgeType(const std::string& name,
                             const std::vector<PropertyDef>& properties,
                             const std::vector<RelationDef>& relations) {
    label_t edge_id = elabel_indexer_.insert(name);

    for (const auto& rel : relations) {
        label_t src_id = get_vertex_label_id(rel.source);
        label_t dst_id = get_vertex_label_id(rel.destination);

        uint32_t triplet_id = generate_edge_label(src_id, dst_id, edge_id);

        auto es = std::make_shared<EdgeSchema>();
        es->src_label_id = src_id;
        es->dst_label_id = dst_id;
        es->edge_label_id = edge_id;
        es->oe_strategy = getOutStrategy(rel.relation);
        es->ie_strategy = getInStrategy(rel.relation);
        // ... set properties

        e_schemas_[triplet_id] = std::move(es);
    }
}
```

### Dropping Types

```cpp
// DDL: DROP VERTEX TYPE
void Schema::DropVertexType(const std::string& name) {
    label_t id = get_vertex_label_id(name);
    vlabel_tomb_.set(id);  // Mark as deleted
    v_schemas_[id].reset();
}

// DDL: DROP EDGE TYPE
void Schema::DropEdgeType(const std::string& name) {
    label_t id = get_edge_label_id(name);

    // Remove all triplets with this edge label
    for (auto& [triplet_id, schema] : e_schemas_) {
        if (schema->edge_label_id == id) {
            elabel_triplet_tomb_.set(triplet_id);
            schema.reset();
        }
    }

    elabel_tomb_.set(id);
}
```

## DDL Statements

### Create Vertex Type

```cypher
CREATE VERTEX TYPE person IF NOT EXISTS {
    id INT64,
    fName STRING,
    lName STRING,
    age INT32,
    PRIMARY KEY (id)
}
```

### Create Edge Type

```cypher
CREATE EDGE TYPE knows IF NOT EXISTS {
    since DATE,
    weight DOUBLE
}
FROM person TO person WITH RELATION many_to_many
```

### Drop Vertex Type

```cypher
DROP VERTEX TYPE person IF EXISTS
```

### Drop Edge Type

```cypher
DROP EDGE TYPE knows IF EXISTS
```

### Alter Vertex Type

```cypher
ALTER VERTEX TYPE person ADD PROPERTY email STRING DEFAULT ""
```

### Alter Edge Type

```cypher
ALTER EDGE TYPE knows ADD PROPERTY note STRING DEFAULT ""
```

## Edge Triplet ID

Edges are uniquely identified by a triplet:

```
(src_label_id, dst_label_id, edge_label_id) → triplet_id
```

This allows the same edge label for different vertex type pairs:

```yaml
edge_types:
  - type_name: connected
    vertex_type_pair_relations:
      - source_vertex: person
        destination_vertex: person
      - source_vertex: company
        destination_vertex: company
```

Creates two edge tables:
- Triplet (person, person, connected)
- Triplet (company, company, connected)

## Schema Evolution

### Supported Operations

| Operation | Support |
|-----------|---------|
| Add vertex type | ✓ |
| Add edge type | ✓ |
| Add property | ✓ |
| Drop vertex type | ✓ (soft delete) |
| Drop edge type | ✓ (soft delete) |
| Drop property | ✓ (soft delete) |
| Rename type | ✗ |
| Change property type | ✗ |
| Change primary key | ✗ |

### Soft Deletion

Dropped types/properties are marked as deleted but not removed:

```cpp
vlabel_tomb_.set(id);  // Mark vertex type as deleted
vprop_soft_deleted[col_idx] = true;  // Mark property as deleted
```

Actual removal happens during compaction.

## File Storage

Schema is persisted to:

```
database/
├── schema.yaml           # Human-readable schema
└── schema.bin            # Binary format for fast loading
```

## GCatalog (GOpt Integration)

For query compilation, schema is wrapped in `GCatalog`:

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
};
```

## References

- [Property Graph Model](./property-graph-model.md)
- [Vertex Table](./vertex-table.md)
- [Edge Table](./edge-table.md)
- [GOpt Framework](../03-query-engine/gopt-framework.md)