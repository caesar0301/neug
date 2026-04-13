# MVCC Model

This document describes NeuG's Multi-Version Concurrency Control implementation.

## Overview

NeuG uses **MVCC (Multi-Version Concurrency Control)** for transaction isolation in Service Mode:

- **Snapshot Isolation**: Each transaction sees a consistent snapshot
- **Timestamp Ordering**: Transactions are ordered by timestamps
- **Multi-version Data**: Multiple versions of data coexist

## Timestamp Model

### Timestamp Types

```cpp
using timestamp_t = uint32_t;

// Special timestamps
constexpr timestamp_t MAX_TIMESTAMP = UINT32_MAX;
constexpr timestamp_t INVALID_TIMESTAMP = 0;
```

### Timestamp Allocation

```cpp
// From version_manager.h
class TPVersionManager {
    std::atomic<timestamp_t> write_ts_;    // Next write timestamp
    std::atomic<timestamp_t> read_ts_;     // Current read watermark

public:
    timestamp_t acquire_read_timestamp() {
        return read_ts_.load();
    }

    timestamp_t acquire_insert_timestamp() {
        return write_ts_.fetch_add(1);
    }

    timestamp_t acquire_update_timestamp() {
        // Wait for all in-flight transactions
        wait_for_pending();
        return write_ts_.fetch_add(1);
    }
};
```

## Visibility Rules

### Version Visibility

A data version is visible to a transaction if:

```
visible(version, read_ts) =
    version.create_ts <= read_ts
    AND (version.delete_ts == 0 OR version.delete_ts > read_ts)
```

### Example

```
Time:  100    150    200    250    300
       │      │      │      │      │
       │      │      │      │      └─ Read at ts=300
       │      │      │      │
       │      │      │      └─ Version 2 deleted at ts=250
       │      │      │
       │      │      └─ Version 2 created at ts=200
       │      │
       │      └─ Version 1 updated at ts=150
       │
       └─ Version 1 created at ts=100

Versions at ts=120:  V1 (visible)
Versions at ts=180:  V1 (visible, but marked deleted)
Versions at ts=220:  V2 (visible)
Versions at ts=280:  V2 (visible, but marked deleted)
```

## Transaction Types and Concurrency

### Concurrency Matrix

|  | Read | Insert | Update |
|--|------|--------|--------|
| **Read** | ✓ Concurrent | ✓ Concurrent | ✗ Blocked |
| **Insert** | ✓ Concurrent | ✓ Concurrent | ✗ Blocked |
| **Update** | ✗ Blocks | ✗ Blocks | ✗ Exclusive |

### Read Transaction

```cpp
class ReadTransaction {
    const PropertyGraph& graph_;
    IVersionManager& vm_;
    timestamp_t timestamp_;

public:
    ReadTransaction(const PropertyGraph& graph, IVersionManager& vm)
        : graph_(graph), vm_(vm) {
        timestamp_ = vm.acquire_read_timestamp();
    }

    ~ReadTransaction() {
        vm_.release_read_timestamp();
    }

    // All reads use timestamp_ for visibility
    VertexSet GetVertexSet(label_t label) {
        return graph_.GetVertexSet(label, timestamp_);
    }
};
```

### Insert Transaction

```cpp
class InsertTransaction {
    PropertyGraph& graph_;
    IVersionManager& vm_;
    timestamp_t timestamp_;
    InArchive wal_archive_;

public:
    InsertTransaction(PropertyGraph& graph, IVersionManager& vm)
        : graph_(graph), vm_(vm) {
        timestamp_ = vm.acquire_insert_timestamp();
    }

    vid_t AddVertex(label_t label, const std::string& oid,
                    const std::vector<Value>& props) {
        // Insert with timestamp_
        vid_t vid = graph_.AddVertex(label, oid, props, timestamp_);
        // Log to WAL
        wal_archive_ << InsertVertexRedo{label, oid, props};
        return vid;
    }

    void Commit() {
        wal_archive_.flush();
        vm_.release_insert_timestamp();
    }
};
```

### Update Transaction

```cpp
class UpdateTransaction {
    PropertyGraph& graph_;
    IVersionManager& vm_;
    timestamp_t timestamp_;
    std::stack<std::unique_ptr<IUndoLog>> undo_logs_;

public:
    UpdateTransaction(PropertyGraph& graph, IVersionManager& vm)
        : graph_(graph), vm_(vm) {
        // Waits for all pending transactions
        timestamp_ = vm.acquire_update_timestamp();
    }

    void UpdateVertexProperty(vid_t vid, size_t col, const Value& val) {
        // Save old value for rollback
        Value old = graph_.GetVertexProperty(vid, col);
        undo_logs_.push(std::make_unique<UpdateVertexUndoLog>(vid, col, old));

        // Apply update
        graph_.UpdateVertexProperty(vid, col, val, timestamp_);
    }

    void Commit() {
        // Clear undo logs (no rollback needed)
        while (!undo_logs_.empty()) undo_logs_.pop();
        vm_.release_update_timestamp();
    }

    void Abort() {
        // Rollback in reverse order
        while (!undo_logs_.empty()) {
            undo_logs_.top()->apply(graph_);
            undo_logs_.pop();
        }
        vm_.release_update_timestamp();
    }
};
```

## Version Storage

### Vertex Version

```cpp
// From vertex_timestamp.h
class VertexTimestamp {
    mmap_array<timestamp_t> create_ts_;
    mmap_array<timestamp_t> delete_ts_;

public:
    bool isVisible(vid_t vid, timestamp_t ts) const {
        return create_ts_[vid] <= ts &&
               (delete_ts_[vid] == 0 || delete_ts_[vid] > ts);
    }
};
```

### Edge Version

```cpp
// From nbr.h
template <typename EDATA_T>
struct ImmutableNbr {
    vid_t neighbor;
    EDATA_T data;
    timestamp_t create_ts;
    timestamp_t delete_ts;  // 0 if not deleted
};

// During iteration
template <typename EDATA_T>
class NbrIterator {
    ImmutableNbr<EDATA_T>* ptr;
    timestamp_t read_ts;

    NbrIterator& operator++() {
        ++ptr;
        // Skip invisible versions
        while (ptr < end && !isVisible(*ptr, read_ts)) {
            ++ptr;
        }
        return *this;
    }
};
```

## Garbage Collection

### Version Cleanup

Old versions are cleaned up during compaction:

```cpp
void CompactTransaction::compact(timestamp_t min_active_ts) {
    // Remove versions deleted before min_active_ts
    for (vid_t vid : vertices) {
        auto& create = create_ts_[vid];
        auto& del = delete_ts_[vid];

        if (del > 0 && del < min_active_ts) {
            // Permanently remove vertex
            remove_vertex(vid);
        }
    }
}
```

### Compaction Trigger

```cpp
// Triggered when:
// 1. Deleted data exceeds threshold
// 2. Explicit CHECKPOINT command
// 3. Database close

void PropertyGraph::Compact() {
    timestamp_t min_ts = vm_.get_min_active_timestamp();
    CompactTransaction tx(*this, min_ts);
    tx.run();
}
```

## Snapshot Isolation

### Characteristics

| Property | Behavior |
|----------|----------|
| **Read Consistency** | All reads see same snapshot |
| **Write Conflicts** | First-committer-wins |
| **Phantom Reads** | Prevented by timestamp ordering |
| **Dirty Reads** | Prevented (only committed data visible) |

### Example Scenario

```
Transaction A (ts=100)          Transaction B (ts=110)
──────────────────────────────────────────────────────
BEGIN
READ n.age = 30
                                BEGIN
                                READ n.age = 30
                                SET n.age = 35
                                COMMIT (ts=110)
SET n.age = 40
COMMIT (ts=120)
                                → Success! A's write overwrites B's
```

## Version Manager Implementation

```cpp
// From version_manager.h
class TPVersionManager : public IVersionManager {
    std::atomic<timestamp_t> write_ts_{1};
    std::atomic<timestamp_t> read_ts_{0};
    std::atomic<int> pending_reads_{0};
    std::atomic<int> pending_inserts_{0};
    Bitset released_ts_;
    SpinLock lock_;

public:
    timestamp_t acquire_read_timestamp() override {
        read_ts_.store(write_ts_.load() - 1);
        pending_reads_.fetch_add(1);
        return read_ts_.load();
    }

    void release_read_timestamp() override {
        pending_reads_.fetch_sub(1);
    }

    timestamp_t acquire_insert_timestamp() override {
        pending_inserts_.fetch_add(1);
        return write_ts_.fetch_add(1);
    }

    void release_insert_timestamp() override {
        pending_inserts_.fetch_sub(1);
    }

    timestamp_t acquire_update_timestamp() override {
        std::unique_lock<SpinLock> guard(lock_);
        // Wait for all pending transactions
        while (pending_reads_.load() > 0 || pending_inserts_.load() > 0) {
            guard.unlock();
            std::this_thread::yield();
            guard.lock();
        }
        return write_ts_.fetch_add(1);
    }

    void release_update_timestamp() override {
        // Update complete, allow others to proceed
    }

    timestamp_t get_min_active_timestamp() {
        return released_ts_.find_first_zero();
    }
};
```

## Performance Considerations

### Timestamp Overhead

| Operation | Overhead |
|-----------|----------|
| Read timestamp | Atomic load (fast) |
| Insert timestamp | Atomic fetch_add (fast) |
| Update timestamp | Wait + atomic (can block) |

### Version Accumulation

Too many versions slow down scans:
- Run `CHECKPOINT` periodically
- Use compaction to clean old versions

### Long-Running Transactions

Long transactions prevent version cleanup:
- Keep transactions short
- Avoid idle transactions

## References

- [Transaction Types](./transaction-types.md)
- [WAL](./wal.md)
- [Version Manager](./version-manager.md)