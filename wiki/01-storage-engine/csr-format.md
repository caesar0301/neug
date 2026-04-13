# CSR Format (Compressed Sparse Row)

This document describes NeuG's CSR-based edge storage.

## Overview

NeuG uses **Compressed Sparse Row (CSR)** format for storing graph edges. CSR is a space-efficient data structure optimized for graph traversal operations.

### Why CSR?

| Benefit | Description |
|---------|-------------|
| **Space Efficient** | O(V + E) space for V vertices and E edges |
| **Cache Friendly** | Contiguous memory layout for neighbor access |
| **Fast Traversal** | O(1) access to neighbor list start, O(degree) iteration |
| **Compression** | Implicit vertex indexing via offsets |

## CSR Structure

### Conceptual Model

```
Graph with 4 vertices and 7 edges:
0 → 1, 0 → 2
1 → 3
3 → 0, 3 → 1, 3 → 2

CSR Representation:
adj_lists:  [1, 2, 3, 0, 1, 2]     # Concatenated neighbor lists
offsets:    [0, 2, 3, 3, 6]         # Start index for each vertex
                                       # offsets[i+1] - offsets[i] = degree
```

### Memory Layout

```
┌─────────────────────────────────────────────────────────────┐
│                     CSR Memory Layout                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  offsets[] (size = V + 1)                                   │
│  ┌─────┬─────┬─────┬─────┬─────┐                           │
│  │  0  │  2  │  3  │  3  │  6  │                           │
│  └─────┴─────┴─────┴─────┴─────┘                           │
│    │     │     │     │     │                                │
│    │     │     │     │     └─ end sentinel                 │
│    │     │     │     └─ vertex 3 offset                    │
│    │     │     └─ vertex 2 offset (degree = 0)             │
│    │     └─ vertex 1 offset                                 │
│    └─ vertex 0 offset                                        │
│                                                             │
│  adj_lists[] (size = E)                                     │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐                     │
│  │  1  │  2  │  3  │  0  │  1  │  2  │                     │
│  └─────┴─────┴─────┴─────┴─────┴─────┘                     │
│    │     │     │     │     │     │                          │
│    └──┬──┘     │     └──┬──┴──┬──┘                          │
│       │        │        │     │                              │
│   v0's nbrs   v1's    v3's nbrs                             │
│   [1,2]       nbr [3]  [0,1,2]                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## NeuG Implementation

### CsrBase Interface

```cpp
// From csr_base.h
class CsrBase {
public:
    virtual ~CsrBase() = default;

    // Get view for iteration
    virtual std::shared_ptr<GenericView> get_generic_view(
        vid_t vid, timestamp_t ts) = 0;

    // Edge operations
    virtual void put_edge(vid_t src, vid_t dst,
                          const Property& data) = 0;
    virtual void batch_put_edges(
        const std::vector<vid_t>& srcs,
        const std::vector<vid_t>& dsts,
        const std::vector<Property>& datas) = 0;

    // Compaction
    virtual void compact() = 0;
};
```

### CSR Types

| Type | Description | Use Case |
|------|-------------|----------|
| `ImmutableCsr` | Read-only, loaded from disk | Batch-loaded graphs |
| `MutableCsr` | Dynamic with per-vertex locks | Insert/update workloads |
| `SingleImmutableCsr` | At most one edge per pair | `one_to_one` edges |
| `SingleMutableCsr` | Single edge, updatable | Mutable `one_to_one` |
| `EmptyCsr` | No storage | `None` edge strategy |

### ImmutableCsr

Optimized for read-heavy workloads:

```cpp
// From immutable_csr.h
template <typename EDATA_T>
class ImmutableCsr : public CsrBase {
    mmap_array<nbr_t*> adj_lists_;    // Neighbor arrays
    mmap_array<int> degree_list_;     // Degree per vertex

    // O(1) neighbor access
    nbr_t* get_neighbors(vid_t vid) {
        return adj_lists_[vid];
    }

    int get_degree(vid_t vid) {
        return degree_list_[vid];
    }
};
```

### MutableCsr

Supports dynamic edge insertion:

```cpp
// From mutable_csr.h
template <typename EDATA_T>
class MutableCsr : public CsrBase {
    SpinLock* locks_;                      // Per-vertex locks
    mmap_array<nbr_t*> adj_list_buffer_;   // Neighbor lists
    mmap_array<std::atomic<int>> adj_list_size_;    // Current size
    mmap_array<int> adj_list_capacity_;             // Buffer capacity

    void put_edge(vid_t src, vid_t dst, const Property& data) {
        locks_[src].lock();
        // Resize if needed, append edge
        locks_[src].unlock();
    }
};
```

### Neighbor Structure

```cpp
// From nbr.h
template <typename EDATA_T>
struct ImmutableNbr {
    vid_t neighbor;    // Destination vertex ID
    EDATA_T data;      // Edge property data
    timestamp_t timestamp;  // MVCC timestamp
};

template <typename EDATA_T>
struct MutableNbr {
    vid_t neighbor;
    EDATA_T data;
    timestamp_t timestamp;
    std::atomic<MutableNbr*> next;  // Linked list for updates
};
```

## GenericView (Iteration Interface)

```cpp
// From generic_view.h
class GenericView {
    nbr_t* adjlists_;
    int* degrees_;
    NbrIterConfig cfg_;
    timestamp_t timestamp_;

public:
    // Get all edges from a vertex (with MVCC filtering)
    NbrList get_edges(vid_t vid) {
        return NbrList(adjlists_ + offsets_[vid],
                       degrees_[vid],
                       timestamp_);
    }
};

// STL-compatible iteration
class NbrList {
    nbr_t* begin_ptr;
    nbr_t* end_ptr;
    timestamp_t ts;

public:
    NbrIterator begin() { return NbrIterator(begin_ptr, ts); }
    NbrIterator end() { return NbrIterator(end_ptr, ts); }
};

// Usage in query execution
for (auto& nbr : view.get_edges(vid)) {
    vid_t neighbor = nbr.neighbor();
    auto edge_data = nbr.data();
    // Process edge...
}
```

## MVCC Integration

### Timestamp Filtering

Edges have timestamps for MVCC visibility:

```cpp
struct ImmutableNbr {
    vid_t neighbor;
    EDATA_T data;
    timestamp_t timestamp;  // When edge was created
};

// During iteration
for (auto& nbr : view.get_edges(vid)) {
    if (nbr.timestamp() > read_timestamp) {
        continue;  // Not visible yet
    }
    if (is_deleted(nbr, read_timestamp)) {
        continue;  // Tombstone
    }
    // Process visible edge
}
```

### Edge Updates

Updates create new versions:

```
Original edge: (v0, v1, data="old", ts=100)

Update at ts=150:
├── Original marked as deleted at ts=150
└── New edge: (v0, v1, data="new", ts=150)

Read at ts=120: sees "old"
Read at ts=200: sees "new"
```

## Edge Strategies

CSR storage is configured per direction:

```cpp
enum class EdgeStrategy {
    kMultiple,   // Multiple edges allowed (MutableCsr)
    kSingle,     // At most one edge (SingleMutableCsr)
    kNone,       // No storage (EmptyCsr)
};
```

### Configuration Example

```yaml
edge_types:
  - type_name: knows
    vertex_type_pair_relations:
      - source_vertex: person
        destination_vertex: person
        relation: many_to_many
```

This creates:
- `out_csr_`: MutableCsr (kMultiple) for outgoing edges
- `in_csr_`: MutableCsr (kMultiple) for incoming edges

For `one_to_many` relation:
- `out_csr_`: MutableCsr (kMultiple)
- `in_csr_`: SingleMutableCsr (kSingle)

For `one_to_one` relation:
- `out_csr_`: SingleMutableCsr (kSingle)
- `in_csr_`: SingleMutableCsr (kSingle)

## Performance Characteristics

### Space Complexity

| Component | Space |
|-----------|-------|
| Offsets | O(V) |
| Neighbor list | O(E) |
| Total | O(V + E) |

### Time Complexity

| Operation | Immutable | Mutable |
|-----------|-----------|---------|
| Get neighbors | O(1) | O(1) |
| Iterate all neighbors | O(degree) | O(degree) |
| Insert edge | N/A | O(1) amortized |
| Delete edge | N/A | O(degree) with lock |
| Check edge existence | O(log degree) | O(degree) |

### Memory-Mapped Files

CSR data uses `mmap_array` for persistence:

```cpp
// From mmap_array.h
template <typename T>
class mmap_array {
    int fd_;
    T* data_;
    size_t size_;

    void open(const std::string& path) {
        fd_ = ::open(path.c_str(), O_RDWR);
        data_ = (T*)mmap(nullptr, size_ * sizeof(T),
                         PROT_READ | PROT_WRITE, MAP_SHARED, fd_, 0);
    }

    // Data persists on disk, loaded on-demand by OS
};
```

## Edge Property Storage

Two modes for edge properties:

### Bundled (Inline)

Properties stored directly in neighbor structure:

```cpp
struct ImmutableNbr {
    vid_t neighbor;
    EdgeProperties data;  // Inline properties
};
```

Best for: Small, fixed-size properties.

### Column-Based

Properties stored in separate table:

```cpp
class EdgeTable {
    std::unique_ptr<CsrBase> out_csr_;
    std::unique_ptr<CsrBase> in_csr_;
    std::unique_ptr<Table> table_;  // Property columns
};
```

Best for: Large or variable-size properties.

## References

- [Property Graph Model](./property-graph-model.md)
- [Edge Table](./edge-table.md)
- [Memory Management](./memory-management.md)