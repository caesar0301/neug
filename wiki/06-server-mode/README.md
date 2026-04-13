# Server Mode Overview

This section documents NeuG's transaction processing server mode.

## Documents

| Document | Description |
|----------|-------------|
| [brpc-service.md](./brpc-service.md) | BRPC HTTP service implementation |
| [session-management.md](./session-management.md) | Session pooling and lifecycle |
| [http-endpoints.md](./http-endpoints.md) | REST API reference |

## Architecture

```
┌─────────────────────────────────────────┐
│              BRPC Server                │
│  (bthread-based concurrent handling)    │
├─────────────────────────────────────────┤
│           NeugDBService                 │
│  (HTTP service wrapper)                 │
├─────────────────────────────────────────┤
│           SessionPool                   │
│  (Thread-safe session management)       │
├─────────────────────────────────────────┤
│           NeugDBSession                 │
│  (Per-request transaction context)      │
├─────────────────────────────────────────┤
│           PropertyGraph                 │
│  (Underlying storage with MVCC)         │
└─────────────────────────────────────────┘
```

## Starting the Server

```python
import neug

db = neug.Database("/path/to/db")
db.serve(port=8080, host="0.0.0.0")
```

## HTTP Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/cypher` | POST | Execute Cypher query |
| `/schema` | GET | Get graph schema (YAML) |
| `/service_status` | GET | Health check |

## Request Format

```json
POST /cypher
Content-Type: application/json

{
  "query": "MATCH (n:person) RETURN n.fName",
  "mode": "read",
  "parameters": {}
}
```

## Response Format

```json
{
  "results": [
    {"n.fName": "Alice"},
    {"n.fName": "Bob"}
  ],
  "elapsed_ms": 12.5
}
```

## Key Classes

| Class | Header | Purpose |
|-------|--------|---------|
| `NeugDBService` | `server/neug_db_service.h` | HTTP service wrapper |
| `NeugDBSession` | `server/neug_db_session.h` | Session-based execution |
| `SessionPool` | `server/session_pool.h` | Session pooling |
| `BrpcServiceManager` | `server/brpc_service_mgr.h` | BRPC server management |

## Concurrency Model

- **bthread**: Lightweight user-space threads for concurrent request handling
- **SessionPool**: Thread-safe session reuse
- **MVCC**: Multiple readers, single writer (update transactions are exclusive)