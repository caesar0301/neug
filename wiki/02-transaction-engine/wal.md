# Write-Ahead Logging (WAL)

This document describes NeuG's Write-Ahead Logging system for durability.

## Overview

WAL ensures **durability** of committed transactions:

1. All modifications are logged before commit
2. Log is flushed to disk before commit returns
3. On crash, log is replayed to recover state

## WAL Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         WAL Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Transaction                                                   │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────────┐                                              │
│   │  InArchive  │  Serialize operations                        │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │ IWalWriter  │  Write to log file                           │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │   fsync()   │  Ensure durability                           │
│   └──────┬──────┘                                              │
│          │                                                      │
│          ▼                                                      │
│   ┌─────────────┐                                              │
│   │  wal/*.log  │  Persistent log files                        │
│   └─────────────┘                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## WAL Format

### Log File Structure

```
database/
└── wal/
    ├── wal_000001.log    # Log segment 1
    ├── wal_000002.log    # Log segment 2
    └── wal.meta         # Metadata (last checkpoint, etc.)
```

### Record Format

Each WAL record has a header and payload:

```cpp
// From wal.h
struct WalHeader {
    timestamp_t timestamp;
    WalRecordType type;
    uint32_t length;
};

enum class WalRecordType {
    INSERT_VERTEX = 1,
    INSERT_EDGE = 2,
    UPDATE_VERTEX_PROP = 3,
    UPDATE_EDGE_PROP = 4,
    DELETE_VERTEX = 5,
    DELETE_EDGE = 6,
    CREATE_VERTEX_TYPE = 7,
    CREATE_EDGE_TYPE = 8,
    DROP_VERTEX_TYPE = 9,
    DROP_EDGE_TYPE = 10,
    CHECKPOINT = 11,
};
```

### Redo Record Structures

```cpp
// Insert Vertex
struct InsertVertexRedo {
    label_t label;
    std::string oid;
    std::vector<Value> properties;
};

// Insert Edge
struct InsertEdgeRedo {
    uint32_t triplet_id;
    vid_t src;
    vid_t dst;
    std::vector<Value> properties;
};

// Update Vertex Property
struct UpdateVertexPropRedo {
    vid_t vid;
    label_t label;
    size_t col_idx;
    Value old_value;  // For potential rollback
    Value new_value;
};

// Delete Vertex
struct RemoveVertexRedo {
    vid_t vid;
    label_t label;
};

// DDL Records
struct CreateVertexTypeRedo {
    std::string name;
    std::vector<PropertyDef> properties;
    std::vector<std::string> primary_keys;
};
```

## IWalWriter Interface

```cpp
// From wal.h
class IWalWriter {
public:
    virtual ~IWalWriter() = default;

    virtual bool open(const std::string& path) = 0;
    virtual void close() = 0;
    virtual void append(const char* data, size_t length) = 0;
    virtual void flush() = 0;
};

// Local file implementation
class LocalWalWriter : public IWalWriter {
    int fd_;
    std::vector<char> buffer_;

public:
    bool open(const std::string& path) override {
        fd_ = ::open(path.c_str(), O_WRONLY | O_APPEND | O_CREAT, 0644);
        return fd_ >= 0;
    }

    void append(const char* data, size_t length) override {
        buffer_.insert(buffer_.end(), data, data + length);
    }

    void flush() override {
        ::write(fd_, buffer_.data(), buffer_.size());
        ::fsync(fd_);
        buffer_.clear();
    }

    void close() override {
        if (fd_ >= 0) {
            ::close(fd_);
        }
    }
};

// No-op implementation (for AP mode)
class DummyWalWriter : public IWalWriter {
public:
    void append(const char*, size_t) override {}
    void flush() override {}
};
```

## IWalParser Interface

```cpp
// From wal.h
class IWalParser {
public:
    virtual ~IWalParser() = default;

    virtual bool open(const std::string& path) = 0;
    virtual std::vector<InsertWalRecord> get_insert_wal(timestamp_t ts) = 0;
    virtual std::vector<UpdateWalRecord> get_update_wals() = 0;
    virtual void close() = 0;
};

class LocalWalParser : public IWalParser {
    int fd_;
    char* data_;
    size_t size_;

public:
    bool open(const std::string& path) override {
        fd_ = ::open(path.c_str(), O_RDONLY);
        // Memory map the file
        struct stat st;
        fstat(fd_, &st);
        size_ = st.st_size;
        data_ = (char*)mmap(nullptr, size_, PROT_READ, MAP_PRIVATE, fd_, 0);
        return true;
    }

    std::vector<InsertWalRecord> get_insert_wal(timestamp_t ts) override {
        std::vector<InsertWalRecord> records;
        char* ptr = data_;
        while (ptr < data_ + size_) {
            WalHeader* header = (WalHeader*)ptr;
            if (header->timestamp >= ts) {
                // Parse record
                records.push_back(parse_record(ptr));
            }
            ptr += sizeof(WalHeader) + header->length;
        }
        return records;
    }
};
```

## WAL in Transactions

### Insert Transaction

```cpp
void InsertTransaction::AddVertex(label_t label, const std::string& oid,
                                   const std::vector<Value>& properties) {
    // 1. Insert into storage
    vid_t vid = graph_.AddVertex(label, oid, properties, timestamp_);

    // 2. Write to WAL
    InsertVertexRedo redo;
    redo.label = label;
    redo.oid = oid;
    redo.properties = properties;

    WalHeader header;
    header.timestamp = timestamp_;
    header.type = WalRecordType::INSERT_VERTEX;
    header.length = redo.serialized_size();

    wal_archive_ << header << redo;
}

void InsertTransaction::Commit() {
    // Flush WAL BEFORE releasing timestamp
    wal_.append(wal_archive_.data(), wal_archive_.size());
    wal_.flush();

    vm_.release_insert_timestamp();
}
```

### Update Transaction

```cpp
void UpdateTransaction::SetVertexProperty(vid_t vid, label_t label,
                                           size_t col, const Value& value) {
    // 1. Save undo
    Value old = graph_.GetVertexProperty(vid, label, col);
    undo_logs_.push(std::make_unique<UpdateVertexUndoLog>(vid, col, old));

    // 2. Apply change
    graph_.SetVertexProperty(vid, label, col, value, timestamp_);

    // 3. Write to WAL
    UpdateVertexPropRedo redo;
    redo.vid = vid;
    redo.label = label;
    redo.col_idx = col;
    redo.new_value = value;

    WalHeader header;
    header.timestamp = timestamp_;
    header.type = WalRecordType::UPDATE_VERTEX_PROP;
    header.length = redo.serialized_size();

    wal_archive_ << header << redo;
}
```

## Recovery Process

### Startup Recovery

```cpp
void PropertyGraph::Recover() {
    // 1. Find all WAL files
    std::vector<std::string> wal_files = list_wal_files();

    // 2. Parse each file
    LocalWalParser parser;
    for (const auto& file : wal_files) {
        parser.open(file);

        // 3. Replay insert records
        auto insert_records = parser.get_insert_wal(0);
        for (const auto& rec : insert_records) {
            AddVertex(rec.label, rec.oid, rec.properties);
        }

        // 4. Replay update records
        auto update_records = parser.get_update_wals();
        for (const auto& rec : update_records) {
            SetVertexProperty(rec.vid, rec.label, rec.col, rec.new_value);
        }

        parser.close();
    }

    // 5. Clear processed WAL files
    for (const auto& file : wal_files) {
        unlink(file.c_str());
    }
}
```

### Recovery After Crash

```
┌─────────────────────────────────────────────────────────────────┐
│                     Recovery Timeline                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Last Checkpoint                                               │
│   │                                                             │
│   │    WAL Records                                              │
│   │    │                                                        │
│   │    │                                                        │
│   │    ▼                                                        │
│   ├────┼──────────────────────────────────────────── Crash!     │
│   │    │                                                        │
│   │    └─ Replay these records                                  │
│   │                                                             │
│   ▼                                                              │
│   Recovery Complete                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Checkpoint

### Purpose

Checkpoint creates a consistent snapshot:
1. Flush all in-memory data to disk
2. Record checkpoint timestamp
3. Truncate WAL before checkpoint

### Implementation

```cpp
void PropertyGraph::Checkpoint() {
    // 1. Block new transactions
    std::lock_guard<std::mutex> lock(checkpoint_mutex_);

    // 2. Acquire update timestamp (waits for pending transactions)
    timestamp_t ts = vm_.acquire_update_timestamp();

    // 3. Dump all data
    Dump();

    // 4. Record checkpoint
    WalHeader header;
    header.timestamp = ts;
    header.type = WalRecordType::CHECKPOINT;
    wal_writer_.append((char*)&header, sizeof(header));
    wal_writer_.flush();

    // 5. Truncate old WAL files
    truncate_wal_before(ts);

    // 6. Release timestamp
    vm_.release_update_timestamp();
}
```

### Usage

```python
# Python
conn.execute("CHECKPOINT", mode="schema")
```

## Performance Considerations

### WAL Buffer Size

```cpp
// Larger buffer = fewer flushes but more data at risk
constexpr size_t WAL_BUFFER_SIZE = 4 * 1024 * 1024;  // 4MB
```

### Sync Options

```cpp
enum class SyncMode {
    SYNC_ALL,       // fsync after every commit (safest, slowest)
    SYNC_PERIODIC,  // fsync every N seconds
    SYNC_NONE,      // Let OS handle (fastest, least durable)
};
```

### Group Commit

Multiple transactions can share a single fsync:

```cpp
void GroupCommitManager::add_transaction(InArchive& archive) {
    std::lock_guard<std::mutex> lock(mutex_);
    buffer_.insert(buffer_.end(), archive.data(), archive.data() + archive.size());
    pending_count_++;

    if (pending_count_ >= GROUP_SIZE) {
        flush();
    }
}
```

## Configuration

### Python

```python
import neug

db = neug.Database(
    "/path/to/db",
    wal_enabled=True,
    wal_sync_mode="sync_all"  # "sync_all", "sync_periodic", "sync_none"
)
```

### YAML Config

```yaml
wal:
  enabled: true
  sync_mode: sync_all
  buffer_size: 4194304
  group_commit_size: 10
```

## References

- [Transaction Types](./transaction-types.md)
- [MVCC Model](./mvcc-model.md)
- [Undo Log](./undo-log.md)