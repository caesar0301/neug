# Vertex Table

This document describes NeuG's vertex storage implementation.

## Overview

`VertexTable` manages storage for all vertices of a single label, including:

- **Primary key to internal ID mapping** (via `LFIndexer`)
- **Property columns** (via column-oriented `Table`)
- **MVCC timestamp tracking** (via `VertexTimestamp`)

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        VertexTable                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     LFIndexer                            │   │
│  │          Primary Key (OID) → Internal ID (VID)           │   │
│  │                                                         │   │
│  │   "person_123" → VID 0                                  │   │
│  │   "person_456" → VID 1                                  │   │
│  │   "person_789" → VID 2                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      Table                               │   │
│  │              (Column-Oriented Properties)               │   │
│  │                                                         │   │
│  │   ┌──────────┬──────────┬──────────┬──────────┐        │   │
│  │   │ Column 0 │ Column 1 │ Column 2 │ Column 3 │        │   │
│  │   │   (id)   │ (fName)  │ (lName)  │  (age)   │        │   │
│  │   ├──────────┼──────────┼──────────┼──────────┤        │   │
│  │   │   123    │  "Alice" │  "Smith" │    30    │ VID 0  │   │
│  │   │   456    │   "Bob"  │ "Johnson"│    25    │ VID 1  │   │
│  │   │   789    │ "Charlie"│  "Brown" │    35    │ VID 2  │   │
│  │   └──────────┴──────────┴──────────┴──────────┘        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  VertexTimestamp                         │   │
│  │                   (MVCC Tracking)                        │   │
│  │                                                         │   │
│  │   ┌─────┬─────┬─────┐                                  │   │
│  │   │ ts0 │ ts1 │ ts2 │  Creation/deletion timestamps    │   │
│  │   └─────┴─────┴─────┘                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### LFIndexer (Lock-Free Indexer)

Maps primary keys (OID) to internal IDs (VID).

```cpp
// From id_indexer.h
template <typename OID_T, typename VID_T = vid_t>
class LFIndexer {
    // Concurrent hash map for OID → VID mapping
    ConcurrentHashMap<OID_T, VID_T> oid_to_vid_;

    // Reverse mapping for OID lookup
    std::vector<OID_T> vid_to_oid_;

public:
    // Lookup OID, return VID
    bool get_index(const OID_T& oid, VID_T& vid) const;

    // Insert new OID, assign VID
    bool insert(const OID_T& oid, VID_T& vid);

    // Get OID from VID
    const OID_T& get_oid(VID_T vid) const;
};
```

**Features**:
- Lock-free reads for high concurrency
- Fine-grained write locks
- O(1) average lookup time

### Table (Column-Oriented Storage)

Stores vertex properties in columnar format.

```cpp
// From table.h
class Table {
    std::vector<std::unique_ptr<ColumnBase>> columns_;
    size_t num_rows_;

public:
    // Add a column
    void addColumn(std::unique_ptr<ColumnBase> column);

    // Get column by index
    ColumnBase* getColumn(size_t idx);

    // Insert row
    void insert(const std::vector<Value>& values);

    // Get value at (row, column)
    Value getValue(size_t row, size_t col) const;
};
```

**Benefits of Columnar Storage**:
- Efficient column scans (e.g., `SUM(age)`)
- Better compression ratios
- Cache-friendly for analytical queries

### VertexTimestamp (MVCC Support)

Tracks creation and deletion times for each vertex.

```cpp
// From vertex_timestamp.h
class VertexTimestamp {
    mmap_array<timestamp_t> create_ts_;   // Creation timestamp
    mmap_array<timestamp_t> delete_ts_;   // Deletion timestamp (0 if not deleted)

public:
    // Check if vertex is visible at timestamp
    bool isVisible(vid_t vid, timestamp_t ts) const {
        return create_ts_[vid] <= ts &&
               (delete_ts_[vid] == 0 || delete_ts_[vid] > ts);
    }

    // Mark vertex as created
    void setCreated(vid_t vid, timestamp_t ts);

    // Mark vertex as deleted
    void setDeleted(vid_t vid, timestamp_t ts);
};
```

## VertexTable Class

```cpp
// From vertex_table.h
class VertexTable {
    using IndexerType = LFIndexer<std::string, vid_t>;

    IndexerType indexer_;                    // OID → VID mapping
    std::unique_ptr<Table> table_;           // Property columns
    VertexTimestamp v_ts_;                   // MVCC timestamps
    label_t label_id_;

public:
    // Add a vertex, return assigned VID
    vid_t AddVertex(const std::string& oid,
                    const std::vector<Value>& properties);

    // Get VID from OID
    bool get_index(const std::string& oid, vid_t& vid) const;

    // Get OID from VID
    const std::string& GetOid(vid_t vid) const;

    // Get property column
    ColumnBase* GetPropertyColumn(size_t col_idx);

    // Check vertex visibility (MVCC)
    bool IsValidVertex(vid_t vid, timestamp_t ts) const;

    // Iterate all visible vertices
    VertexSet GetVertexSet(timestamp_t ts) const;
};
```

## Operations

### Vertex Insertion

```cpp
vid_t VertexTable::AddVertex(const std::string& oid,
                              const std::vector<Value>& properties) {
    vid_t vid;

    // 1. Insert into indexer, get new VID
    if (!indexer_.insert(oid, vid)) {
        throw VertexAlreadyExists(oid);
    }

    // 2. Insert properties into table
    table_->insert(properties);

    // 3. Set creation timestamp
    v_ts_.setCreated(vid, current_timestamp_);

    return vid;
}
```

### Vertex Lookup

```cpp
// Lookup by primary key
bool VertexTable::get_index(const std::string& oid, vid_t& vid) const {
    return indexer_.get_index(oid, vid);
}

// Lookup by internal ID (for iteration)
const std::string& VertexTable::GetOid(vid_t vid) const {
    return indexer_.get_oid(vid);
}
```

### Property Access

```cpp
// Get single property
Value VertexTable::GetProperty(vid_t vid, size_t col_idx) const {
    return table_->getValue(vid, col_idx);
}

// Get property column (for batch operations)
ColumnBase* VertexTable::GetPropertyColumn(size_t col_idx) {
    return table_->getColumn(col_idx);
}
```

### Vertex Deletion

```cpp
void VertexTable::DeleteVertex(vid_t vid, timestamp_t ts) {
    // Soft delete - mark with timestamp
    v_ts_.setDeleted(vid, ts);

    // Actual removal happens during compaction
}
```

### Vertex Iteration (MVCC-aware)

```cpp
class VertexSet {
    vid_t start_vid_;
    vid_t end_vid_;
    timestamp_t ts_;
    const VertexTimestamp* v_ts_;

public:
    class Iterator {
        vid_t current_;
        timestamp_t ts_;
        const VertexTimestamp* v_ts_;

    public:
        vid_t operator*() const { return current_; }

        Iterator& operator++() {
            do {
                ++current_;
            } while (current_ < end_ && !v_ts_->isVisible(current_, ts_));
            return *this;
        }
    };

    Iterator begin();
    Iterator end();
};

// Usage
VertexSet vertices = vertex_table.GetVertexSet(read_timestamp);
for (vid_t vid : vertices) {
    // Process visible vertex
}
```

## Storage Layout

### File Structure

```
database/
├── vertex_person/
│   ├── indexer.bin       # OID → VID mapping
│   ├── table/
│   │   ├── column_0.bin  # Primary key column
│   │   ├── column_1.bin  # Property 1
│   │   └── column_2.bin  # Property 2
│   └── timestamps.bin    # MVCC timestamps
└── vertex_company/
    ├── indexer.bin
    └── ...
```

### Memory Mapping

All data uses `mmap_array` for persistence:

```cpp
// Data is memory-mapped, not loaded into heap
mmap_array<timestamp_t> create_ts_;
create_ts_.open("vertex_person/timestamps.bin");

// Access like normal array (OS handles paging)
timestamp_t ts = create_ts_[vid];
```

## Primary Key Types

Supported primary key types:

| Type | Storage | Comparison |
|------|---------|------------|
| `INT32` | 4 bytes | Numeric |
| `INT64` | 8 bytes | Numeric |
| `STRING` | Variable | Lexicographic |

### String Primary Keys

String OIDs are stored in a separate buffer:

```cpp
class LFIndexer<std::string, vid_t> {
    ConcurrentHashMap<std::string, vid_t> oid_to_vid_;
    mmap_array<char> string_buffer_;      // Pooled strings
    mmap_array<size_t> string_offsets_;   // Offsets into buffer
};
```

## Performance Considerations

### Bulk Loading

For bulk insertion, use `COPY FROM`:

```cypher
COPY person FROM 'persons.csv' WITH (BATCH_SIZE=10000)
```

This bypasses individual insert overhead and uses optimized batch loading.

### Index Caching

The `LFIndexer` caches frequently accessed OIDs:

```cpp
// Cache hit is O(1)
// Cache miss requires hash table lookup
```

### Column Pruning

For queries accessing few columns:

```cypher
MATCH (n:person) RETURN n.fName  -- Only need 1 column
```

Only the required column is loaded into memory.

## References

- [Property Graph Model](./property-graph-model.md)
- [Schema Management](./schema-management.md)
- [ID Indexer](../10-internals/id-indexer.md)