# Edge Table

This document describes NeuG's edge storage implementation.

## Overview

`EdgeTable` manages storage for edges of a specific label triplet (src_label, dst_label, edge_label):

- **CSR structures** for outgoing and incoming edges
- **Property storage** for edge data
- **MVCC support** for concurrent access

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        EdgeTable                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Label Triplet: (person, person, knows)                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Outgoing CSR                          │   │
│  │           (Edges where this vertex is source)           │   │
│  │                                                         │   │
│  │   Vertex 0: [edge→1, edge→2]                           │   │
│  │   Vertex 1: [edge→3]                                   │   │
│  │   Vertex 2: []                                         │   │
│  │   Vertex 3: [edge→0]                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Incoming CSR                          │   │
│  │          (Edges where this vertex is destination)       │   │
│  │                                                         │   │
│  │   Vertex 0: [edge←3]                                   │   │
│  │   Vertex 1: [edge←0]                                   │   │
│  │   Vertex 2: [edge←0]                                   │   │
│  │   Vertex 3: [edge←1]                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 Property Table                           │   │
│  │               (Edge Properties)                         │   │
│  │                                                         │   │
│  │   ┌──────────┬──────────┬──────────┐                   │   │
│  │   │  since   │  weight  │  note    │                   │   │
│  │   ├──────────┼──────────┼──────────┤                   │   │
│  │   │2020-01-01│   0.95   │ "friend" │  Edge 0→1        │   │
│  │   │2019-06-15│   0.80   │ "colleague"│ Edge 0→2       │   │
│  │   │2021-03-20│   0.90   │ "family" │  Edge 1→3        │   │
│  │   └──────────┴──────────┴──────────┘                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## EdgeTable Class

```cpp
// From edge_table.h
class EdgeTable {
    std::unique_ptr<CsrBase> out_csr_;     // Outgoing edges CSR
    std::unique_ptr<CsrBase> in_csr_;      // Incoming edges CSR
    std::unique_ptr<Table> table_;         // Edge property table

    label_t edge_label_id_;
    label_t src_label_id_;
    label_t dst_label_id_;

public:
    // Edge operations
    void AddEdge(vid_t src, vid_t dst,
                 const std::vector<Value>& properties);

    void BatchAddEdges(const std::vector<vid_t>& srcs,
                       const std::vector<vid_t>& dsts,
                       const std::vector<std::vector<Value>>& properties);

    void DeleteEdge(vid_t src, vid_t dst, int32_t oe_offset,
                    int32_t ie_offset, timestamp_t ts);

    // Edge traversal
    GenericView get_outgoing_view(vid_t vid, timestamp_t ts);
    GenericView get_incoming_view(vid_t vid, timestamp_t ts);

    // Property access
    EdgeDataAccessor get_edge_data_accessor();
};
```

## Edge Operations

### Add Edge

```cpp
void EdgeTable::AddEdge(vid_t src, vid_t dst,
                        const std::vector<Value>& properties) {
    // 1. Insert properties into table
    size_t row_idx = table_->insert(properties);

    // 2. Add to outgoing CSR
    out_csr_->put_edge(src, dst, row_idx);

    // 3. Add to incoming CSR
    in_csr_->put_edge(dst, src, row_idx);
}
```

### Batch Add Edges

```cpp
void EdgeTable::BatchAddEdges(const std::vector<vid_t>& srcs,
                               const std::vector<vid_t>& dsts,
                               const std::vector<std::vector<Value>>& properties) {
    // 1. Batch insert properties
    std::vector<size_t> row_indices = table_->batch_insert(properties);

    // 2. Batch add to CSRs (more efficient than individual inserts)
    out_csr_->batch_put_edges(srcs, dsts, row_indices);

    // For incoming, swap src/dst
    in_csr_->batch_put_edges(dsts, srcs, row_indices);
}
```

### Delete Edge

```cpp
void EdgeTable::DeleteEdge(vid_t src, vid_t dst,
                           int32_t oe_offset, int32_t ie_offset,
                           timestamp_t ts) {
    // Soft delete - mark edge with tombstone
    out_csr_->mark_deleted(src, oe_offset, ts);
    in_csr_->mark_deleted(dst, ie_offset, ts);

    // Actual removal happens during compaction
}
```

## Edge Traversal

### Outgoing Edges

```cpp
// Get view for outgoing edges
GenericView view = edge_table.get_outgoing_view(src_vid, read_timestamp);

// Iterate neighbors
for (auto& nbr : view.get_edges(src_vid)) {
    vid_t dst = nbr.neighbor();
    auto& data = nbr.data();
    // Process edge
}
```

### Incoming Edges

```cpp
// Get view for incoming edges
GenericView view = edge_table.get_incoming_view(dst_vid, read_timestamp);

// Iterate neighbors
for (auto& nbr : view.get_edges(dst_vid)) {
    vid_t src = nbr.neighbor();  // Source vertex
    // Process edge
}
```

## Edge Strategies

Edge storage is configured per direction:

```cpp
enum class EdgeStrategy {
    kMultiple,   // Multiple edges per vertex pair
    kSingle,     // At most one edge per vertex pair
    kNone,       // No storage (skip this direction)
};
```

### Strategy Selection

| Relation Type | Outgoing Strategy | Incoming Strategy |
|---------------|-------------------|-------------------|
| `many_to_many` | `kMultiple` | `kMultiple` |
| `many_to_one` | `kMultiple` | `kSingle` |
| `one_to_many` | `kSingle` | `kMultiple` |
| `one_to_one` | `kSingle` | `kSingle` |

### CSR Type by Strategy

| Strategy | CSR Type |
|----------|----------|
| `kMultiple` | `MutableCsr` or `ImmutableCsr` |
| `kSingle` | `SingleMutableCsr` or `SingleImmutableCsr` |
| `kNone` | `EmptyCsr` (no storage) |

### Example: kNone Strategy

For `one_to_many` edges, incoming edges use `kMultiple` (any vertex can have multiple incoming edges), but we might choose `kNone` for incoming to save space:

```cpp
// Only store outgoing edges, skip incoming
EdgeSchema schema;
schema.oe_strategy = EdgeStrategy::kMultiple;
schema.ie_strategy = EdgeStrategy::kNone;
```

Query impact:
- `(a)-[r]->(b)` - fast (uses outgoing CSR)
- `(a)<-[r]-(b)` - slow (requires full scan)

## Edge Properties

### Bundled Storage

Properties stored inline in CSR neighbor:

```cpp
template <typename EDATA_T>
struct ImmutableNbr {
    vid_t neighbor;
    EDATA_T data;  // Inline properties (struct)
};
```

Best for: Small, fixed-size properties.

### Column Storage

Properties stored in separate table, referenced by index:

```cpp
struct ImmutableNbr {
    vid_t neighbor;
    size_t row_idx;  // Index into property table
};
```

Best for: Large or variable-size properties.

### Property Access

```cpp
// Get edge property via accessor
EdgeDataAccessor accessor = edge_table.get_edge_data_accessor();

for (auto& nbr : view.get_edges(vid)) {
    // Get property by name
    auto since = nbr.data().get<date_t>("since");

    // Or by column index
    auto weight = nbr.data().get<double>(1);
}
```

## Edge Triplet ID

Each edge table is identified by a triplet ID:

```cpp
// From schema.h
uint32_t Schema::generate_edge_label(label_t src, label_t dst, label_t edge) {
    // Unique ID for (src_label, dst_label, edge_label) combination
    return hash(src, dst, edge);
}
```

This allows:
- Same edge label for different vertex type pairs
- Efficient lookup by any component

### Example

```yaml
edge_types:
  - type_name: knows
    vertex_type_pair_relations:
      - source_vertex: person
        destination_vertex: person
      - source_vertex: robot
        destination_vertex: robot
```

Creates two edge tables:
- `(person, person, knows)` → triplet_id = X
- `(robot, robot, knows)` → triplet_id = Y

## MVCC Support

### Edge Visibility

```cpp
struct ImmutableNbr {
    vid_t neighbor;
    EDATA_T data;
    timestamp_t create_ts;
    timestamp_t delete_ts;  // 0 if not deleted
};

// Check visibility
bool isVisible(timestamp_t read_ts) const {
    return create_ts <= read_ts &&
           (delete_ts == 0 || delete_ts > read_ts);
}
```

### Edge Updates

```cpp
void EdgeTable::UpdateEdge(vid_t src, vid_t dst,
                           const std::vector<Value>& new_properties,
                           timestamp_t ts) {
    // 1. Find existing edge
    auto& nbr = find_edge(src, dst);

    // 2. Mark old version as deleted
    nbr.delete_ts = ts;

    // 3. Insert new version
    AddEdge(src, dst, new_properties, ts);
}
```

## File Layout

```
database/
├── edge_person_person_knows/
│   ├── out_csr/
│   │   ├── adj_lists.bin     # Neighbor lists
│   │   ├── offsets.bin       # Vertex offsets
│   │   └── degrees.bin       # Degrees
│   ├── in_csr/
│   │   ├── adj_lists.bin
│   │   ├── offsets.bin
│   │   └── degrees.bin
│   └── table/
│       ├── column_0.bin      # since (date)
│       └── column_1.bin      # weight (double)
```

## Performance Characteristics

### Edge Insertion

| Operation | Complexity |
|-----------|------------|
| Single insert | O(1) amortized |
| Batch insert | O(E) |
| Delete | O(degree) to find, O(1) to mark |

### Edge Traversal

| Operation | Complexity |
|-----------|------------|
| Get outgoing edges | O(1) |
| Get incoming edges | O(1) |
| Iterate neighbors | O(degree) |
| Check edge exists | O(log degree) sorted, O(degree) unsorted |

### Space Usage

| Component | Space |
|-----------|-------|
| Outgoing CSR | O(E_out) |
| Incoming CSR | O(E_in) |
| Property table | O(E × property_size) |
| Total | O(E) |

## References

- [CSR Format](./csr-format.md)
- [Property Graph Model](./property-graph-model.md)
- [Transaction Types](../02-transaction-engine/transaction-types.md)