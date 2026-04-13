# Transaction Engine Overview

This section documents NeuG's transaction management system, including MVCC, WAL, and transaction types.

## Documents

| Document | Description |
|----------|-------------|
| [mvcc-model.md](./mvcc-model.md) | Multi-Version Concurrency Control implementation |
| [transaction-types.md](./transaction-types.md) | Read, Insert, Update, Compact transactions |
| [wal.md](./wal.md) | Write-Ahead Logging for durability |
| [version-manager.md](./version-manager.md) | Timestamp management and visibility |
| [undo-log.md](./undo-log.md) | Rollback mechanism for aborts |

## Transaction Model

NeuG supports two transaction modes:

| Mode | Concurrency | Isolation |
|------|-------------|-----------|
| **Embedded (AP)** | Global lock | Single-threaded |
| **Service (TP)** | MVCC | Snapshot isolation |

## Transaction Types

| Type | Operations | Concurrency |
|------|------------|-------------|
| `ReadTransaction` | Read-only queries | Concurrent with all |
| `InsertTransaction` | Bulk inserts | Concurrent with reads/inserts |
| `UpdateTransaction` | Read/write/DDL | Exclusive (blocks others) |
| `CompactTransaction` | Background cleanup | Exclusive |

## Key Classes

| Class | Header | Purpose |
|-------|--------|---------|
| `ReadTransaction` | `transaction/read_transaction.h` | Snapshot reads |
| `InsertTransaction` | `transaction/insert_transaction.h` | Bulk inserts with WAL |
| `UpdateTransaction` | `transaction/update_transaction.h` | Full CRUD with MVCC |
| `TPVersionManager` | `transaction/version_manager.h` | Timestamp management |
| `LocalWalWriter` | `transaction/wal/local_wal_writer.h` | WAL persistence |

## MVCC Visibility

```
Read Timestamp: 100
├── Visible:   ts <= 100 and not deleted at ts <= 100
└── Not Visible: ts > 100 or deleted at ts <= 100
```

## WAL Structure

```
WalHeader { timestamp, type, length }
├── InsertVertexRedo
├── InsertEdgeRedo
├── UpdateVertexPropRedo
├── UpdateEdgePropRedo
├── RemoveVertexRedo
├── RemoveEdgeRedo
└── DDL Redo records
```