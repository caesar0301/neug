# Transaction Types

This document describes NeuG's transaction types and their characteristics.

## Overview

NeuG supports four transaction types, each optimized for different workloads:

| Type | Operations | Concurrency | Use Case |
|------|------------|-------------|----------|
| `ReadTransaction` | Read-only | High | Queries, analytics |
| `InsertTransaction` | Insert only | Medium | Bulk loading |
| `UpdateTransaction` | Full CRUD + DDL | Low | Updates, schema changes |
| `CompactTransaction` | Cleanup | Exclusive | Maintenance |

## Transaction Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                   Transaction Lifecycle                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────┐     ┌──────────┐     ┌─────────┐                │
│   │  Begin  │────▶│  Work    │────▶│ Commit  │                │
│   └─────────┘     └──────────┘     └─────────┘                │
│        │               │               ▲                       │
│        │               │               │                       │
│        │               ▼               │                       │
│        │         ┌──────────┐          │                       │
│        │         │  Abort   │──────────┘                       │
│        │         └──────────┘                                  │
│        │                                                        │
│        ▼                                                        │
│   Acquire Timestamp                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## ReadTransaction

### Characteristics

- **Read-only** operations
- **Snapshot isolation** at acquired timestamp
- **Concurrent** with other reads and inserts
- **No WAL** needed

### API

```cpp
// From read_transaction.h
class ReadTransaction {
public:
    ReadTransaction(const PropertyGraph& graph, IVersionManager& vm);
    ~ReadTransaction();

    // Vertex operations
    VertexSet GetVertexSet(label_t label);
    bool GetVertexIndex(label_t label, const std::string& oid, vid_t& vid);

    // Property access
    Value GetVertexProperty(vid_t vid, label_t label, size_t col);
    ColumnBase* GetVertexPropertyColumn(label_t label, size_t col);

    // Edge operations
    GenericView GetOutgoingEdges(vid_t vid, uint32_t triplet_id);
    GenericView GetIncomingEdges(vid_t vid, uint32_t triplet_id);

    // Timestamp
    timestamp_t GetTimestamp() const { return timestamp_; }

private:
    const PropertyGraph& graph_;
    IVersionManager& vm_;
    timestamp_t timestamp_;
};
```

### Usage

```python
# Python
conn = db.connect()
result = conn.execute("MATCH (n:person) RETURN n.name", mode="read")
```

```cpp
// C++
auto tx = graph.CreateReadTransaction();
auto vertices = tx.GetVertexSet(person_label);
for (vid_t vid : vertices) {
    auto name = tx.GetVertexProperty(vid, person_label, name_col);
}
tx.Commit();
```

### Implementation

```cpp
ReadTransaction::ReadTransaction(const PropertyGraph& graph, IVersionManager& vm)
    : graph_(graph), vm_(vm) {
    timestamp_ = vm.acquire_read_timestamp();
}

ReadTransaction::~ReadTransaction() {
    vm_.release_read_timestamp();
}

void ReadTransaction::Commit() {
    // No-op for read transactions
    vm_.release_read_timestamp();
}
```

## InsertTransaction

### Characteristics

- **Insert-only** operations
- **No reads** of uncommitted data
- **Concurrent** with reads and other inserts
- **WAL** for durability

### API

```cpp
// From insert_transaction.h
class InsertTransaction {
public:
    InsertTransaction(PropertyGraph& graph, IVersionManager& vm, IWalWriter& wal);
    ~InsertTransaction();

    // Vertex insertion
    vid_t AddVertex(label_t label, const std::string& oid,
                    const std::vector<Value>& properties);

    // Edge insertion
    void AddEdge(label_t edge_label, vid_t src, vid_t dst,
                 const std::vector<Value>& properties);

    // Batch operations
    void BatchAddVertices(label_t label,
                          const std::vector<std::string>& oids,
                          const std::vector<std::vector<Value>>& properties);

    void BatchAddEdges(label_t edge_label,
                       const std::vector<vid_t>& srcs,
                       const std::vector<vid_t>& dsts,
                       const std::vector<std::vector<Value>>& properties);

    // Commit/Abort
    void Commit();
    void Abort();

    timestamp_t GetTimestamp() const { return timestamp_; }

private:
    PropertyGraph& graph_;
    IVersionManager& vm_;
    IWalWriter& wal_;
    timestamp_t timestamp_;
    InArchive wal_archive_;
    std::vector<std::unique_ptr<IdIndexerBase<vid_t>>> added_vertices_;
};
```

### Usage

```python
# Python - Bulk insert
conn.execute("""
    UNWIND [{id: 1, name: 'Alice'}, {id: 2, name: 'Bob'}] AS row
    CREATE (n:person {id: row.id, name: row.name})
""", mode="insert")
```

```cpp
// C++
auto tx = graph.CreateInsertTransaction();
tx.AddVertex(person_label, "person_1", {123, "Alice"});
tx.AddEdge(knows_label, vid1, vid2, {date("2024-01-01")});
tx.Commit();
```

### Implementation

```cpp
vid_t InsertTransaction::AddVertex(label_t label, const std::string& oid,
                                    const std::vector<Value>& properties) {
    // 1. Insert into graph
    vid_t vid = graph_.AddVertex(label, oid, properties, timestamp_);

    // 2. Track for rollback
    added_vertices_[label]->insert(oid, vid);

    // 3. Log to WAL
    InsertVertexRedo redo;
    redo.label = label;
    redo.oid = oid;
    redo.properties = properties;
    wal_archive_ << redo;

    return vid;
}

void InsertTransaction::Commit() {
    // Flush WAL
    wal_.append(wal_archive_.data(), wal_archive_.size());
    wal_.flush();

    // Release timestamp
    vm_.release_insert_timestamp();
}

void InsertTransaction::Abort() {
    // Remove inserted vertices
    for (auto& [label, indexer] : added_vertices_) {
        for (auto& [oid, vid] : *indexer) {
            graph_.RemoveVertex(label, vid);
        }
    }

    vm_.release_insert_timestamp();
}
```

## UpdateTransaction

### Characteristics

- **Full CRUD** operations
- **DDL** operations (create/drop types)
- **Exclusive** access (blocks all other transactions)
- **WAL** for durability
- **Undo log** for rollback

### API

```cpp
// From update_transaction.h
class UpdateTransaction {
public:
    UpdateTransaction(PropertyGraph& graph, IVersionManager& vm, IWalWriter& wal);
    ~UpdateTransaction();

    // Read operations
    VertexSet GetVertexSet(label_t label);
    Value GetVertexProperty(vid_t vid, label_t label, size_t col);
    GenericView GetOutgoingEdges(vid_t vid, uint32_t triplet_id);

    // Insert operations
    vid_t AddVertex(label_t label, const std::string& oid,
                    const std::vector<Value>& properties);
    void AddEdge(label_t edge_label, vid_t src, vid_t dst,
                 const std::vector<Value>& properties);

    // Update operations
    void SetVertexProperty(vid_t vid, label_t label, size_t col,
                           const Value& value);
    void SetEdgeProperty(uint32_t triplet_id, vid_t src, vid_t dst,
                         size_t col, const Value& value);

    // Delete operations
    void DeleteVertex(vid_t vid, label_t label);
    void DeleteEdge(uint32_t triplet_id, vid_t src, vid_t dst);

    // DDL operations
    void CreateVertexType(const std::string& name,
                          const std::vector<PropertyDef>& properties,
                          const std::vector<std::string>& primary_keys);
    void CreateEdgeType(const std::string& name,
                        const std::vector<PropertyDef>& properties,
                        const std::vector<RelationDef>& relations);
    void DropVertexType(const std::string& name);
    void DropEdgeType(const std::string& name);
    void AddVertexProperty(const std::string& type, const PropertyDef& prop);
    void AddEdgeProperty(const std::string& type, const PropertyDef& prop);

    // Commit/Abort
    void Commit();
    void Abort();

    timestamp_t GetTimestamp() const { return timestamp_; }

private:
    PropertyGraph& graph_;
    IVersionManager& vm_;
    IWalWriter& wal_;
    timestamp_t timestamp_;
    InArchive wal_archive_;
    std::stack<std::unique_ptr<IUndoLog>> undo_logs_;
    std::unordered_set<label_t> deleted_vertex_labels_;
    execution::LocalQueryCache& pipeline_cache_;
};
```

### Usage

```python
# Python
conn.execute("""
    MATCH (n:person {id: 123})
    SET n.age = n.age + 1
    RETURN n.age
""", mode="update")
```

```cpp
// C++
auto tx = graph.CreateUpdateTransaction();
auto vid = tx.GetVertexIndex(person_label, "person_123");
tx.SetVertexProperty(vid, person_label, age_col, Value(31));
tx.Commit();
```

### Undo Log

```cpp
// From undo_log.h
class IUndoLog {
public:
    virtual ~IUndoLog() = default;
    virtual void apply(PropertyGraph& graph) = 0;
};

class UpdateVertexUndoLog : public IUndoLog {
    vid_t vid_;
    size_t col_;
    Value old_value_;

public:
    void apply(PropertyGraph& graph) override {
        graph.SetVertexProperty(vid_, col_, old_value_);
    }
};

class InsertVertexUndoLog : public IUndoLog {
    label_t label_;
    vid_t vid_;

public:
    void apply(PropertyGraph& graph) override {
        graph.RemoveVertex(label_, vid_);
    }
};
```

### Implementation

```cpp
void UpdateTransaction::SetVertexProperty(vid_t vid, label_t label,
                                           size_t col, const Value& value) {
    // 1. Save old value for rollback
    Value old = graph_.GetVertexProperty(vid, label, col);
    undo_logs_.push(std::make_unique<UpdateVertexUndoLog>(vid, col, old));

    // 2. Apply update
    graph_.SetVertexProperty(vid, label, col, value, timestamp_);

    // 3. Log to WAL
    UpdateVertexPropRedo redo;
    redo.vid = vid;
    redo.col = col;
    redo.value = value;
    wal_archive_ << redo;
}

void UpdateTransaction::Abort() {
    // Rollback in reverse order
    while (!undo_logs_.empty()) {
        undo_logs_.top()->apply(graph_);
        undo_logs_.pop();
    }
    vm_.release_update_timestamp();
}
```

## CompactTransaction

### Characteristics

- **Cleanup** operations
- **Exclusive** access
- **Background** task

### API

```cpp
// From compact_transaction.h
class CompactTransaction {
public:
    CompactTransaction(PropertyGraph& graph, timestamp_t min_active_ts);

    void CompactVertices();
    void CompactEdges(uint32_t triplet_id);

private:
    PropertyGraph& graph_;
    timestamp_t min_active_ts_;
};
```

### Usage

```python
# Python
conn.execute("CHECKPOINT", mode="schema")
```

## Access Mode Selection

### Python API

```python
# Read mode (default)
conn.execute("MATCH (n) RETURN n", mode="read")
conn.execute("MATCH (n) RETURN n", mode="r")  # Shorthand

# Insert mode
conn.execute("CREATE (n:person {id: 1})", mode="insert")
conn.execute("CREATE (n:person {id: 1})", mode="i")

# Update mode
conn.execute("MATCH (n) SET n.x = 1", mode="update")
conn.execute("MATCH (n) SET n.x = 1", mode="u")

# Schema mode (DDL)
conn.execute("CREATE VERTEX TYPE test {}", mode="schema")
conn.execute("CREATE VERTEX TYPE test {}", mode="s")
```

### Java API

```java
// Read mode
session.run("MATCH (n) RETURN n", Values.parameters(), AccessMode.READ);

// Insert mode
session.run("CREATE (n)", Values.parameters(), AccessMode.INSERT);

// Update mode
session.run("SET n.x = 1", Values.parameters(), AccessMode.UPDATE);

// Schema mode
session.run("CREATE VERTEX TYPE test {}", Values.parameters(), AccessMode.SCHEMA);
```

## References

- [MVCC Model](./mvcc-model.md)
- [WAL](./wal.md)
- [Undo Log](./undo-log.md)