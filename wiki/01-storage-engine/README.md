# Storage Engine Overview

This section documents NeuG's storage layer implementation, including the property graph model, CSR format, and data management.

## Documents

| Document | Description |
|----------|-------------|
| [property-graph-model.md](./property-graph-model.md) | Vertices, edges, properties, labels, primary keys |
| [hypergraph-considerations.md](./hypergraph-considerations.md) | Why hypergraphs are not supported, future possibilities |
| [csr-format.md](./csr-format.md) | Compressed Sparse Row format for edge storage |
| [vertex-table.md](./vertex-table.md) | Vertex storage and primary key to ID mapping |
| [edge-table.md](./edge-table.md) | Edge storage with CSR structures |
| [schema-management.md](./schema-management.md) | YAML schema format and label registration |
| [memory-management.md](./memory-management.md) | mmap_array and memory levels |

## Storage Architecture

```
PropertyGraph
├── Schema (vertex/edge definitions)
├── VertexTables[] (per label)
│   ├── LFIndexer (PK → VID mapping)
│   ├── Table (columnar properties)
│   └── VertexTimestamp (MVCC)
└── EdgeTables{} (by label triplet)
    ├── MutableCsr / ImmutableCsr (outgoing)
    ├── MutableCsr / ImmutableCsr (incoming)
    └── Table (edge properties)
```

## Key Classes

| Class | Header | Purpose |
|-------|--------|---------|
| `PropertyGraph` | `storages/graph/property_graph.h` | Main storage engine |
| `Schema` | `storages/graph/schema.h` | Graph type definitions |
| `VertexTable` | `storages/graph/vertex_table.h` | Per-label vertex storage |
| `EdgeTable` | `storages/graph/edge_table.h` | CSR-based edge storage |
| `MutableCsr` | `storages/csr/mutable_csr.h` | Dynamic CSR for writes |
| `ImmutableCsr` | `storages/csr/immutable_csr.h` | Read-optimized CSR |

## CSR Format

NeuG uses **Compressed Sparse Row (CSR)** format for efficient edge storage:

- **Space-efficient**: O(V + E) space for V vertices and E edges
- **Cache-friendly**: Contiguous memory layout for neighbor iteration
- **Direction-aware**: Separate CSR for outgoing and incoming edges

## Edge Strategies

| Strategy | Description |
|----------|-------------|
| `kMultiple` | Multiple edges allowed between same vertex pair |
| `kSingle` | At most one edge between same vertex pair |
| `kNone` | No edges stored in this direction |