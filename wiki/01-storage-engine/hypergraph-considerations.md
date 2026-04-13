# Hypergraph Considerations

## Current Status: Not Supported

NeuG implements a **standard property graph model** with binary edges only. Each edge connects exactly two vertices: a source and a destination.

### What is a Hypergraph?

A hypergraph is a generalization of a graph where an edge (called a **hyperedge**) can connect any number of vertices, not just two. Hyperedges can represent:

- Multi-way relationships (e.g., a meeting with multiple participants)
- Group memberships (e.g., users in multiple groups)
- N-ary relationships (e.g., a transaction involving buyer, seller, item, and payment)

### Why NeuG Doesn't Support Hypergraphs

#### 1. Edge Schema is Binary

The `EdgeSchema` definition explicitly supports only source-destination pairs:

```cpp
// include/neug/storages/graph/schema.h
struct EdgeSchema {
  std::string src_label_name;  // Single source vertex type
  std::string dst_label_name;  // Single destination vertex type
  std::string edge_label_name;
  // No support for multiple endpoint vertices
};
```

#### 2. CSR Storage Format

The Compressed Sparse Row (CSR) format stores edges as adjacency lists:

```cpp
// include/neug/storages/csr/nbr.h
template <typename EDATA_T>
struct ImmutableNbr {
  vid_t neighbor;  // Single neighbor vertex ID
  EDATA_T data;    // Edge properties
};
```

Each neighbor entry references exactly one vertex, making the format incompatible with hyperedges.

#### 3. Edge Operations

All edge operations assume binary relationships:

```cpp
// include/neug/storages/graph/edge_table.h
void BatchAddEdges(
    const std::vector<vid_t>& src_lid_list,  // Source vertices
    const std::vector<vid_t>& dst_lid_list,  // Destination vertices
    ...
);
```

#### 4. Protocol Buffer Schema

The schema definition in `proto/schema.proto` encodes edges as label pairs:

```protobuf
message RelationMeta {
  message LabelPair {
    LabelMeta src = 1;  // Single source
    LabelMeta dst = 2;  // Single destination
  }
  // No repeated field for multiple endpoints
}
```

## Workarounds for Hyperedge-like Scenarios

### 1. Intermediate Entity Pattern

Model the hyperedge as an intermediate vertex:

```
# Instead of hyperedge connecting A, B, C directly:
(A)-[PARTICIPATES]->(Meeting)<-[PARTICIPATES]-(B)
                          |
                     [PARTICIPATES]
                          |
                         (C)
```

### 2. Property Lists

Store additional vertex references as properties:

```cypher
CREATE (m:Meeting {
  participants: ["Alice", "Bob", "Charlie"]
})
```

Note: This loses graph traversal capabilities for the participant relationship.

### 3. Multiple Binary Edges

Create multiple edges to represent group membership:

```cypher
CREATE (g:Group {name: "Engineering"})
CREATE (a:Person {name: "Alice"})
CREATE (b:Person {name: "Bob"})
CREATE (a)-[:MEMBER_OF]->(g)
CREATE (b)-[:MEMBER_OF]->(g)
```

## Future Considerations

Adding hypergraph support would require significant architectural changes:

| Component | Changes Needed |
|-----------|----------------|
| **Schema** | New `HyperedgeSchema` with `repeated LabelMeta endpoints` |
| **CSR Format** | Modified structure to store sets of neighbors |
| **Query Language** | Cypher extensions for hyperedge patterns |
| **Execution** | New join algorithms for hyperedge matching |
| **Protobuf** | New message types for hyperedge encoding |

### Potential API (Hypothetical)

```cypher
# Hypothetical hyperedge syntax
CREATE (a:Person)-[r:MEETING {time: "2024-01-15"}]->(b:Person), (c:Person)
WHERE type(r) = "hyperedge" AND r.endpoints = [a, b, c]
```

## References

- Property Graph Model: [property-graph-model.md](./property-graph-model.md)
- CSR Format: [csr-format.md](./csr-format.md)
- Edge Table: [edge-table.md](./edge-table.md)