# NeuG Technical Wiki

Welcome to the NeuG technical documentation wiki. This wiki provides comprehensive documentation of NeuG's core engine architecture, storage systems, and development guides.

## Quick Navigation

| Section | Description |
|---------|-------------|
| [00-architecture](./00-architecture/) | System design, dual-mode architecture, data flow |
| [01-storage-engine](./01-storage-engine/) | Property graph model, CSR format, vertex/edge tables |
| [02-transaction-engine](./02-transaction-engine/) | MVCC, WAL, transaction types, version management |
| [03-query-engine](./03-query-engine/) | Compilation pipeline, GOpt framework, optimizer rules |
| [04-execution-engine](./04-execution-engine/) | Pipeline execution, context columns, operators |
| [05-client-apis](./05-client-apis/) | Python, Java, C++ APIs, Cypher manual |
| [06-server-mode](./06-server-mode/) | BRPC service, session management, HTTP endpoints |
| [07-data-management](./07-data-management/) | Data loading, export, built-in datasets |
| [08-extensions](./08-extensions/) | Extension development, JSON, Parquet support |
| [09-development](./09-development/) | Building, testing, contributing |
| [10-internals](./10-internals/) | ID indexer, serialization, memory-mapped utilities |

## About NeuG

**NeuG** (Network-unified Graph) is a high-performance graph database for HTAP (Hybrid Transactional/Analytical Processing) workloads developed by Alibaba's GraphScope team.

### Key Features

- **Dual-Mode Architecture**: Embedded mode for analytics, Service mode for transactions
- **Cypher-Native**: Industry-standard Cypher query language with GOpt optimization
- **CSR Storage**: Efficient Compressed Sparse Row format for graph topology
- **MVCC + WAL**: Full ACID transactions with write-ahead logging
- **Extensible**: Postgres/DuckDB-inspired extension system

### Data Model

NeuG uses a **standard property graph model** with binary edges:
- **Vertices**: Labeled entities with properties and primary keys
- **Edges**: Directed relationships connecting exactly two vertices (source → destination)
- **Properties**: Key-value pairs on vertices and edges

> **Note**: NeuG does not currently support hypergraphs (edges connecting multiple vertices). See [hypergraph-considerations.md](./01-storage-engine/hypergraph-considerations.md) for details.

## Documentation Structure

```
wiki/
├── 00-architecture/       # System-level design documents
├── 01-storage-engine/     # Storage layer deep-dives
├── 02-transaction-engine/ # Concurrency and durability
├── 03-query-engine/       # Query compilation and optimization
├── 04-execution-engine/   # Runtime execution
├── 05-client-apis/        # User-facing APIs
├── 06-server-mode/        # Transaction processing server
├── 07-data-management/    # Data I/O operations
├── 08-extensions/         # Extension framework
├── 09-development/        # Developer guides
└── 10-internals/          # Low-level implementation details
```

## Getting Started

1. New to NeuG? Start with [system-design.md](./00-architecture/system-design.md)
2. Understanding data storage? Read [property-graph-model.md](./01-storage-engine/property-graph-model.md)
3. Building queries? See [compilation-pipeline.md](./03-query-engine/compilation-pipeline.md)
4. Using the API? Check [python-api.md](./05-client-apis/python-api.md) or [java-driver.md](./05-client-apis/java-driver.md)

## Contributing

See [contributing.md](./09-development/contributing.md) for how to contribute to this documentation.