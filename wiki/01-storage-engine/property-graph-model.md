# Property Graph Model

This document describes NeuG's property graph data model.

## Overview

NeuG implements a **standard property graph model** with:

- **Vertices**: Labeled entities with properties
- **Edges**: Directed relationships between exactly two vertices
- **Properties**: Key-value pairs attached to vertices and edges

> **Note**: NeuG does NOT support hypergraphs. See [hypergraph-considerations.md](./hypergraph-considerations.md) for details.

## Core Concepts

### Vertex

A vertex represents an entity in the graph.

| Attribute | Description |
|-----------|-------------|
| **Label** | Type/category (e.g., `person`, `company`) |
| **Internal ID** | Unique integer assigned by NeuG |
| **Primary Key** | User-defined unique identifier |
| **Properties** | Key-value pairs |

### Edge

An edge represents a directed relationship between two vertices.

| Attribute | Description |
|-----------|-------------|
| **Label** | Type of relationship (e.g., `knows`, `works_at`) |
| **Source Vertex** | Where the edge originates |
| **Destination Vertex** | Where the edge points to |
| **Properties** | Key-value pairs |

### Property

Key-value pairs attached to vertices or edges.

| Type | Examples |
|------|----------|
| `BOOL` | `true`, `false` |
| `INT32` | `-2147483648` to `2147483647` |
| `INT64` | Large integers |
| `FLOAT` | 32-bit floating point |
| `DOUBLE` | 64-bit floating point |
| `STRING` | UTF-8 strings |
| `DATE` | Date values |
| `TIMESTAMP` | Date-time values |
| `LIST<T>` | Homogeneous lists |
| `STRUCT` | Nested structures |

## Schema Definition

### YAML Format

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
        - property_name: age
          property_type: INT32

    - type_name: company
      type_id: 1
      primary_keys: [name]
      properties:
        - property_name: name
          property_type: STRING
        - property_name: revenue
          property_type: DOUBLE

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
```

### Edge Multiplicity

| Relation Type | Description |
|---------------|-------------|
| `one_to_one` | At most one edge between any vertex pair |
| `one_to_many` | One source can have multiple edges, but each destination has at most one |
| `many_to_one` | Multiple sources can point to same destination, but each source has at most one edge |
| `many_to_many` | No restrictions on edge count |

## Internal Representation

### Vertex IDs

NeuG uses two ID systems:

| ID Type | Description |
|---------|-------------|
| **OID (Object ID)** | User-defined primary key (e.g., `person.id`) |
| **VID (Vertex ID)** | Internal consecutive integer (0, 1, 2, ...) |

The mapping is maintained by `LFIndexer`:

```cpp
// From vertex_table.h
class VertexTable {
    LFIndexer indexer_;  // Maps OID → VID
    std::unique_ptr<Table> table_;  // Property columns
};
```

### Edge Storage

Edges are stored in CSR (Compressed Sparse Row) format:

```
Outgoing Edges CSR:
Vertex 0: [edge→1, edge→2]
Vertex 1: [edge→3]
Vertex 2: []
Vertex 3: [edge→0, edge→1, edge→2]
```

See [csr-format.md](./csr-format.md) for details.

## Schema Classes

### PropertyGraph

Main storage engine class.

```cpp
// From property_graph.h
class PropertyGraph {
    Schema schema_;
    std::vector<VertexTable> vertex_tables_;
    std::unordered_map<uint32_t, EdgeTable> edge_tables_;

    // Key methods
    void Open(const std::string& work_dir);
    void Dump(bool reopen = true);
    void CreateVertexType(label_t label, const VertexSchema& schema);
    void CreateEdgeType(label_t label, const EdgeSchema& schema);
};
```

### Schema

Manages vertex and edge type definitions.

```cpp
// From schema.h
class Schema {
    IdIndexer<std::string, label_t> vlabel_indexer_;  // Label name → ID
    IdIndexer<std::string, label_t> elabel_indexer_;
    std::vector<std::shared_ptr<VertexSchema>> v_schemas_;
    std::unordered_map<uint32_t, std::shared_ptr<EdgeSchema>> e_schemas_;

    // Key methods
    label_t get_vertex_label_id(const std::string& name);
    label_t get_edge_label_id(const std::string& name);
    uint32_t generate_edge_label(label_t src, label_t dst, label_t edge);
};
```

### VertexSchema

Defines a single vertex type.

```cpp
// From schema.h
struct VertexSchema {
    std::string label_name;
    std::vector<std::string> primary_keys;
    std::vector<DataType> property_types;
    std::vector<std::string> property_names;
    std::vector<std::string> default_property_values;
    size_t max_num;  // Maximum vertex count
};
```

### EdgeSchema

Defines a single edge type between a vertex pair.

```cpp
// From schema.h
struct EdgeSchema {
    std::string src_label_name;
    std::string dst_label_name;
    std::string edge_label_name;
    EdgeStrategy oe_strategy;  // Outgoing: Multiple/Single/None
    EdgeStrategy ie_strategy;  // Incoming: Multiple/Single/None
    std::vector<DataType> property_types;
    std::vector<std::string> property_names;
};
```

## Edge Triplet ID

Edges are uniquely identified by a **triplet ID**:

```
edge_triplet_id = hash(src_label, dst_label, edge_label)
```

This allows:
- Same edge label for different vertex pairs
- Efficient lookup by source, destination, or edge label

## Cypher Mapping

### Creating Vertices

```cypher
CREATE (n:person {id: 123, fName: "Alice", age: 30})
```

Creates:
1. Entry in `vertex_tables_[0].indexer_`: `123 → VID`
2. Property row in `vertex_tables_[0].table_`

### Creating Edges

```cypher
MATCH (a:person {id: 123}), (b:person {id: 456})
CREATE (a)-[r:knows {since: date("2020-01-01")}]->(b)
```

Creates:
1. Edge in `edge_tables_[triplet_id].out_csr_` at vertex `a`
2. Edge in `edge_tables_[triplet_id].in_csr_` at vertex `b`
3. Property row in `edge_tables_[triplet_id].table_`

### Querying

```cypher
MATCH (n:person {id: 123})-[:knows]->(m:person)
RETURN n.fName, m.fName
```

Execution:
1. Lookup `123` in `vertex_tables_[0].indexer_` → `VID`
2. Access `edge_tables_[knows_triplet].out_csr_` at `VID`
3. For each neighbor, access `vertex_tables_[0].table_` for properties

## Limitations

### Binary Edges Only

Each edge connects exactly two vertices:

```
✓ (a)-[r]->(b)     # Supported
✗ (a)-[r]-(b)-(c)  # Not supported (hyperedge)
```

Workaround: Use intermediate vertices for multi-way relationships.

### No Edge Primary Keys

Edges are identified by (source, destination, label) triplet:

```cypher
# Multiple edges between same vertices require many_to_many relation
CREATE (a)-[r1:knows]->(b)  # Edge 1
CREATE (a)-[r2:knows]->(b)  # Edge 2 (only if relation is many_to_many)
```

### Single Primary Key

Vertices support only one primary key column:

```yaml
# Valid
primary_keys: [id]

# Invalid
primary_keys: [first_name, last_name]
```

## References

- [CSR Format](./csr-format.md)
- [Vertex Table](./vertex-table.md)
- [Edge Table](./edge-table.md)
- [Schema Management](./schema-management.md)