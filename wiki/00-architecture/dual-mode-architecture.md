# Dual-Mode Architecture

NeuG supports two operational modes optimized for different workloads.

## Overview

| Mode | Purpose | Concurrency | Transaction Model |
|------|---------|-------------|-------------------|
| **Embedded** | Analytics, batch processing | Single-threaded | Global lock |
| **Service** | Real-time transactions | Multi-user | MVCC |

## Embedded Mode (AP)

### Characteristics

- **AP**: Analytical Processing
- **Single-threaded execution**
- **Global lock** for data consistency
- **Optimized for bulk operations**

### When to Use

- Data science workflows
- ML/AI pipelines
- Graph analytics algorithms
- Bulk data loading
- Offline batch processing

### Usage

```python
import neug

# Embedded mode - single user
db = neug.Database("/path/to/db")
conn = db.connect()

# Bulk load data
conn.execute("COPY person FROM 'persons.csv'")

# Run analytics
result = conn.execute("""
    MATCH (a:person)-[:knows*1..3]->(b:person)
    RETURN a.id, b.id
""")

# Close when done
conn.close()
```

### Performance Characteristics

| Operation | Performance |
|-----------|-------------|
| Bulk Loading | Very fast (no MVCC overhead) |
| Complex Queries | Fast (no lock contention) |
| Pattern Matching | Optimized for large graphs |
| Graph Algorithms | Efficient (single-threaded) |

## Service Mode (TP)

### Characteristics

- **TP**: Transaction Processing
- **Multi-user concurrent access**
- **MVCC** for transaction isolation
- **Session pooling** for efficiency

### When to Use

- Real-time applications
- Web services
- Concurrent users
- OLTP workloads
- Interactive queries

### Usage

```python
import neug

db = neug.Database("/path/to/db")

# Start server
db.serve(port=8080, host="0.0.0.0")

# Now clients can connect via HTTP
# POST /cypher with JSON body
```

### Client Connection

```python
import neug

# Remote connection
session = neug.Session.open("http://localhost:8080")

# Execute queries
result = session.execute("""
    MATCH (n:person {id: $id})
    SET n.visits = n.visits + 1
    RETURN n.visits
""", parameters={"id": 123}, mode="update")

session.close()
```

### HTTP API

```bash
# Execute query
curl -X POST http://localhost:8080/cypher \
  -H "Content-Type: application/json" \
  -d '{"query": "MATCH (n:person) RETURN n.fName LIMIT 10", "mode": "read"}'

# Get schema
curl http://localhost:8080/schema

# Health check
curl http://localhost:8080/service_status
```

## Switching Modes

### Embedded → Service

```python
db = neug.Database("/path/to/db")
conn = db.connect()

# Do embedded work
conn.execute("LOAD json")
conn.close()  # MUST close connection first

# Switch to service mode
db.serve(port=8080)
```

### Service → Embedded

```python
# Service must be stopped first
# (currently requires process restart)

db = neug.Database("/path/to/db")
db.serve(port=8080)
# ... serve requests ...

# To switch back to embedded, restart process
```

## Transaction Model Comparison

| Aspect | Embedded Mode | Service Mode |
|--------|---------------|--------------|
| **Isolation** | Implicit (single thread) | Snapshot isolation |
| **Durability** | Optional checkpoint | WAL enabled |
| **Concurrency** | None | MVCC with timestamps |
| **Lock Granularity** | Global | Row-level (vertices/edges) |

### Embedded Mode Transactions

```python
# Implicit transaction per query
conn.execute("CREATE (n:person {id: 1})")  # Auto-commit

# Explicit multi-statement (via update mode)
conn.execute("""
    MATCH (n:person {id: 1})
    SET n.name = 'Alice'
""", mode="update")
```

### Service Mode Transactions

```python
# Each request gets a session with MVCC
# Read transactions: snapshot at timestamp T
# Write transactions: exclusive lock, new timestamp

# Concurrent reads - allowed
# Concurrent inserts - allowed
# Update transaction - exclusive (blocks others)
```

## Architecture Differences

### Embedded Mode

```
┌─────────────────────────────────┐
│        Python Process           │
│  ┌───────────────────────────┐  │
│  │      NeuG Database        │  │
│  │  ┌─────────────────────┐  │  │
│  │  │   PropertyGraph     │  │  │
│  │  │   (Global Lock)     │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### Service Mode

```
┌─────────────────────────────────────────┐
│            BRPC Server                  │
│  ┌─────────────────────────────────┐    │
│  │         NeugDBService           │    │
│  │  ┌───────────────────────────┐  │    │
│  │  │      SessionPool          │  │    │
│  │  │  ┌─────┐ ┌─────┐ ┌─────┐  │  │    │
│  │  │  │ S1  │ │ S2  │ │ S3  │  │  │    │
│  │  │  └─────┘ └─────┘ └─────┘  │  │    │
│  │  └───────────────────────────┘  │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │        VersionManager           │    │
│  │    (MVCC Timestamp Control)     │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │        PropertyGraph            │    │
│  │    (Concurrent Read Access)     │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

## Performance Considerations

### Embedded Mode Optimization

```python
# Batch loading - very fast
conn.execute("COPY person FROM 'large_file.csv'")

# Complex queries - no lock overhead
result = conn.execute("""
    MATCH p = (a:person)-[:knows*1..5]->(b:person)
    RETURN p
""")

# Checkpoint for durability
conn.execute("CHECKPOINT", mode="schema")
```

### Service Mode Optimization

```python
# Use connection pooling
# Reuse sessions when possible

# Batch inserts in single transaction
session.execute("""
    UNWIND [{id: 1, name: 'A'}, {id: 2, name: 'B'}] AS row
    CREATE (n:person {id: row.id, name: row.name})
""", mode="insert")

# Use appropriate access mode
session.execute("MATCH...", mode="read")      # Concurrent reads
session.execute("CREATE...", mode="insert")   # Concurrent inserts
session.execute("SET...", mode="update")      # Exclusive update
```

## References

- [Transaction Types](../02-transaction-engine/transaction-types.md)
- [MVCC Model](../02-transaction-engine/mvcc-model.md)
- [HTTP Endpoints](../06-server-mode/http-endpoints.md)