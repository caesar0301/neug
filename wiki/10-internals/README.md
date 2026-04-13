# Internals Overview

This section documents NeuG's low-level implementation details.

## Documents

| Document | Description |
|----------|-------------|
| [id-indexer.md](./id-indexer.md) | Lock-free ID indexing (LFIndexer) |
| [serialization.md](./serialization.md) | InArchive/OutArchive for WAL |
| [mmap-utilities.md](./mmap-utilities.md) | Memory-mapped file utilities |
| [performance-considerations.md](./performance-considerations.md) | Performance tips and trade-offs |

## Core Data Types

| Type | Definition | Purpose |
|------|------------|---------|
| `vid_t` | `uint32_t` | Internal vertex ID |
| `label_t` | `uint32_t` | Vertex/edge label ID |
| `timestamp_t` | `uint32_t` | MVCC timestamp |
| `eid_t` | `std::pair<vid_t, vid_t>` | Edge identifier |

## Key Utilities

### IdIndexer / LFIndexer

Lock-free concurrent hash map for ID mapping:

```
PK → Internal ID mapping
- O(1) lookup
- Lock-free reads
- Fine-grained write locks
```

### mmap_array

Memory-mapped file array:

```
- Direct file mapping (no copy)
- OS-managed paging
- Persistence without serialization
```

### InArchive / OutArchive

Binary serialization:

```
- Used for WAL records
- Network protocol encoding
- Type-safe serialization
```

## Memory Management

| Level | Description |
|-------|-------------|
| `kLowMemory` | Minimal memory, aggressive disk usage |
| `kMediumMemory` | Balanced memory/disk |
| `kHighMemory` | Maximum memory for performance |

## Performance Tips

1. **Batch Operations**: Use `COPY FROM` instead of individual `CREATE`
2. **CSR Benefits**: Prefer edge traversal over vertex lookups
3. **Query Caching**: Reuse prepared statements
4. **Appropriate Mode**: Use embedded for analytics, service for transactions

## Key Files

| File | Purpose |
|------|---------|
| `utils/id_indexer.h` | Lock-free ID indexer |
| `utils/mmap_array.h` | Memory-mapped array |
| `utils/serialization/` | Archive serialization |
| `utils/property/` | Property type system |