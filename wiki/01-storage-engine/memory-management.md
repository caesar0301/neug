# Memory Management

This document describes NeuG's memory management system.

## Overview

NeuG uses **memory-mapped files** (`mmap`) for efficient storage and persistence:

- Data persists on disk
- OS manages paging between memory and disk
- No explicit serialization for most operations

## mmap_array

The core data structure for memory-mapped storage.

### Definition

```cpp
// From mmap_array.h
template <typename T>
class mmap_array {
    int fd_ = -1;
    T* data_ = nullptr;
    size_t size_ = 0;
    bool read_only_ = false;

public:
    // Open existing file
    bool open(const std::string& path, bool read_only = false);

    // Create new file
    bool create(const std::string& path, size_t size);

    // Close and unmap
    void close();

    // Access like array
    T& operator[](size_t idx) { return data_[idx]; }
    const T& operator[](size_t idx) const { return data_[idx]; }

    // Iterators
    T* begin() { return data_; }
    T* end() { return data_ + size_; }

    // Size
    size_t size() const { return size_; }

    // Flush to disk
    void sync();
};
```

### Usage Example

```cpp
// Create memory-mapped array
mmap_array<int> degrees;
degrees.create("vertex_degrees.bin", num_vertices);

// Write (goes to memory, OS syncs to disk)
degrees[0] = 5;
degrees[1] = 3;

// Explicit flush
degrees.sync();

// Close
degrees.close();

// Reopen later
degrees.open("vertex_degrees.bin");
int d = degrees[0];  // Read from persisted data
```

### Memory Benefits

| Benefit | Description |
|---------|-------------|
| **Lazy Loading** | OS loads pages on-demand |
| **Page Sharing** | Multiple processes can share same file |
| **No Copy** | Direct file mapping, no heap allocation |
| **Persistence** | Data survives process restart |

## Memory Levels

NeuG supports three memory levels:

```cpp
enum class MemoryLevel {
    kLowMemory,      // Minimal RAM, aggressive disk usage
    kMediumMemory,   // Balanced
    kHighMemory,     // Maximum RAM, cache everything
};
```

### Configuration

```python
import neug

# Specify memory level
db = neug.Database("/path/to/db", memory_level="high")
```

### Impact on Operations

| Memory Level | CSR Loading | Property Caching | Query Performance |
|--------------|-------------|------------------|-------------------|
| Low | On-demand | Minimal | Slower, less RAM |
| Medium | Partial | Moderate | Balanced |
| High | Pre-load | Aggressive | Fast, more RAM |

## Arena Allocator

For temporary allocations during operations:

```cpp
// From arena_allocator.h
class ArenaAllocator {
    std::vector<char*> blocks_;
    size_t current_offset_;
    size_t block_size_;

public:
    // Allocate (very fast, no individual frees)
    void* allocate(size_t size);

    // Reset all at once (no destructor calls)
    void reset();

    // Free all blocks
    void clear();
};
```

### Usage Pattern

```cpp
ArenaAllocator arena;

// During edge insertion
for (auto& edge : edges) {
    // Allocate temp buffer
    char* buf = arena.allocate(edge.size());
    // Process...
}

// Reset after batch
arena.reset();  // Fast - no individual deallocations
```

## Thread-Local Allocators

Per-thread memory pools to avoid contention:

```cpp
// From thread_local_allocator.h
thread_local ArenaAllocator tls_arena;

void* tls_alloc(size_t size) {
    return tls_arena.allocate(size);
}

void tls_reset() {
    tls_arena.reset();
}
```

## Page-Aligned Allocation

For mmap compatibility:

```cpp
// Aligned allocation
void* aligned_alloc(size_t size, size_t alignment = 4096) {
    void* ptr;
    posix_memalign(&ptr, alignment, size);
    return ptr;
}
```

## Memory-Mapped CSR

CSR structures use mmap for persistence:

```cpp
// From immutable_csr.h
template <typename EDATA_T>
class ImmutableCsr {
    mmap_array<nbr_t*> adj_lists_;    // Neighbor arrays
    mmap_array<int> degree_list_;     // Degrees

    void open(const std::string& path) {
        adj_lists_.open(path + "/adj_lists.bin");
        degree_list_.open(path + "/degrees.bin");
    }
};
```

### Layout on Disk

```
edge_person_person_knows/
├── out_csr/
│   ├── adj_lists.bin     # mmap_array<nbr_t*>
│   └── degrees.bin       # mmap_array<int>
└── in_csr/
    ├── adj_lists.bin
    └── degrees.bin
```

## Property Column Storage

Column data is memory-mapped:

```cpp
// From column.h
template <typename T>
class Column {
    mmap_array<T> data_;

    void open(const std::string& path) {
        data_.open(path);
    }

    T& operator[](size_t idx) {
        return data_[idx];
    }
};
```

## WAL Buffer

Write-Ahead Log uses in-memory buffer:

```cpp
// From wal_writer.h
class LocalWalWriter {
    int fd_;
    std::vector<char> buffer_;
    size_t buffer_size_;

public:
    void append(const char* data, size_t len) {
        if (buffer_.size() + len > buffer_size_) {
            flush();
        }
        buffer_.insert(buffer_.end(), data, data + len);
    }

    void flush() {
        write(fd_, buffer_.data(), buffer_.size());
        buffer_.clear();
        fsync(fd_);
    }
};
```

## Query Execution Memory

### Context Columns

Execution context uses columnar data:

```cpp
// From context.h
class Context {
    std::vector<std::shared_ptr<IContextColumn>> columns;

    // Memory is managed per-column
    // Columns can be sparse or dense
};
```

### Column Types

| Type | Memory Layout |
|------|---------------|
| `VertexColumn` | Array of `vid_t` |
| `EdgeColumn` | Array of edge references |
| `ValueColumn<T>` | Array of type T |
| `ListColumn` | Array of pointers + buffers |
| `PathColumn` | Array of path structures |

### Memory Reuse

Columns are reused when possible:

```cpp
// During projection
void ProjectOperator::Eval(Context& ctx) {
    // Reuse existing column if types match
    if (ctx.columns[idx]->type() == result_type) {
        // In-place update
    } else {
        // Allocate new column
        ctx.columns[idx] = createColumn(result_type);
    }
}
```

## Performance Considerations

### Avoiding Page Faults

```cpp
// Prefault pages for sequential access
void prefault(const void* addr, size_t len) {
    madvise(addr, len, MADV_SEQUENTIAL);
    for (size_t i = 0; i < len; i += 4096) {
        volatile char c = ((char*)addr)[i];
    }
}
```

### Memory Pressure

```cpp
// Hint to OS about access patterns
void set_access_pattern(void* addr, size_t len, Pattern pattern) {
    switch (pattern) {
        case SEQUENTIAL:
            madvise(addr, len, MADV_SEQUENTIAL);
            break;
        case RANDOM:
            madvise(addr, len, MADV_RANDOM);
            break;
        case WILLNEED:
            madvise(addr, len, MADV_WILLNEED);
            break;
    }
}
```

### Large Graph Handling

For graphs larger than memory:

1. Use `kLowMemory` level
2. Access vertices sequentially when possible
3. Batch operations to reduce random access
4. Consider edge direction for CSR access

## Memory Statistics

```cpp
struct MemoryStats {
    size_t total_mapped;      // Total mmap'd memory
    size_t resident;          // Actually in RAM
    size_t shared;            // Shared with other processes

    static MemoryStats get() {
        MemoryStats stats;
        // Read from /proc/self/smaps
        return stats;
    }
};
```

## References

- [CSR Format](./csr-format.md)
- [Vertex Table](./vertex-table.md)
- [Edge Table](./edge-table.md)
- [Performance Considerations](../10-internals/performance-considerations.md)