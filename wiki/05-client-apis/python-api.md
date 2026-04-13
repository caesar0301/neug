# Python API Reference

This document provides a comprehensive reference for NeuG's Python API.

## Installation

```bash
pip install neug
```

Requirements:
- Python 3.8+
- Linux, macOS (x86_64, ARM64), or Windows (WSL2)

## Module Overview

```python
import neug

# Main classes
neug.Database      # Database management
neug.Connection    # Query execution
neug.Session       # Remote session (TP mode)
neug.QueryResult   # Result iteration
neug.AsyncConnection  # Async query execution
```

## Database Class

### Constructor

```python
class Database:
    def __init__(
        self,
        db_path: str,
        mode: str = "read_write",  # "read_only" or "read_write"
        memory_level: str = "medium",  # "low", "medium", "high"
        max_thread_num: int = 0  # 0 = auto
    )
```

### Methods

| Method | Description |
|--------|-------------|
| `connect()` | Create connection for embedded mode |
| `async_connect()` | Create async connection |
| `serve(port, host)` | Start server mode |
| `close()` | Close database |
| `load_builtin_dataset(name)` | Load built-in dataset |
| `get_schema()` | Get YAML schema |

### Examples

```python
import neug

# Open database
db = neug.Database("/path/to/db")

# Load built-in dataset
db.load_builtin_dataset("tinysnb")

# Create connection
conn = db.connect()

# Execute queries
result = conn.execute("MATCH (n:person) RETURN n.fName")

# Close
conn.close()
db.close()
```

### Server Mode

```python
# Start server
db.serve(port=8080, host="0.0.0.0")

# Server runs until process terminates
# Handle requests via HTTP API or Session
```

## Connection Class

### Methods

| Method | Description |
|--------|-------------|
| `execute(query, mode, parameters)` | Execute Cypher query |
| `get_schema()` | Get graph schema |
| `close()` | Close connection |

### Access Modes

| Mode | Alias | Description |
|------|-------|-------------|
| `"read"` | `"r"` | Read-only queries |
| `"insert"` | `"i"` | Bulk inserts |
| `"update"` | `"u"` | Read/write queries |
| `"schema"` | `"s"` | DDL operations |

### Examples

```python
# Read query
result = conn.execute("""
    MATCH (n:person)-[r:knows]->(m:person)
    RETURN n.fName, m.fName
""", mode="read")

# Insert
result = conn.execute("""
    CREATE (n:person {id: 123, fName: 'Alice', age: 30})
""", mode="insert")

# Update
result = conn.execute("""
    MATCH (n:person {id: 123})
    SET n.age = 31
""", mode="update")

# Parameterized query
result = conn.execute("""
    MATCH (n:person)
    WHERE n.id = $id
    RETURN n.fName
""", parameters={"id": 123}, mode="read")
```

## QueryResult Class

### Methods

| Method | Description |
|--------|-------------|
| `__iter__()` | Iterate over rows |
| `__next__()` | Get next row |
| `close()` | Close result |
| `get_schema()` | Get result schema |

### Row Access

```python
result = conn.execute("MATCH (n:person) RETURN n.id, n.fName, n.age")

# Iterate
for row in result:
    print(row[0])  # n.id
    print(row[1])  # n.fName
    print(row[2])  # n.age

# With column names
for row in result:
    print(row["n.id"])
    print(row["n.fName"])
```

### Data Types

| Cypher Type | Python Type |
|-------------|-------------|
| `BOOL` | `bool` |
| `INT32` | `int` |
| `INT64` | `int` |
| `FLOAT` | `float` |
| `DOUBLE` | `float` |
| `STRING` | `str` |
| `DATE` | `datetime.date` |
| `TIMESTAMP` | `datetime.datetime` |
| `LIST` | `list` |
| `MAP` | `dict` |

## Session Class

Remote session for connecting to NeuG server.

### Methods

```python
class Session:
    @staticmethod
    def open(
        endpoint: str,      # e.g., "http://localhost:8080"
        timeout: int = 30   # Connection timeout in seconds
    ) -> Session

    def execute(
        self,
        query: str,
        mode: str = "read",
        parameters: dict = None
    ) -> QueryResult

    def close(self)
```

### Example

```python
import neug

# Connect to remote server
session = neug.Session.open("http://localhost:8080")

# Execute queries
result = session.execute("""
    MATCH (n:person) RETURN n.fName
""")

for row in result:
    print(row[0])

session.close()
```

## AsyncConnection Class

Asynchronous query execution.

### Example

```python
import neug
import asyncio

async def main():
    db = neug.Database("/path/to/db")
    conn = await db.async_connect()

    # Async query
    result = await conn.execute("""
        MATCH (n:person) RETURN n.fName
    """)

    async for row in result:
        print(row[0])

    await conn.close()
    db.close()

asyncio.run(main())
```

## Dataset Loading

### Built-in Datasets

```python
db = neug.Database("/path/to/db")

# Load tinysnb (small social network)
db.load_builtin_dataset("tinysnb")

# Available datasets
# - "tinysnb": Small social network
# - "ldbc": LDBC Social Network Benchmark
```

### Custom Data Loading

```python
conn = db.connect()

# CSV
conn.execute("""
    COPY person FROM '/path/to/person.csv'
    WITH (FORMAT='csv', DELIMITER=',', HEADER_ROW=true)
""", mode="insert")

# JSON (requires json extension)
conn.execute("INSTALL json")
conn.execute("LOAD json")
conn.execute("""
    COPY person FROM '/path/to/person.jsonl'
    WITH (FORMAT='json')
""", mode="insert")

# Parquet (requires parquet extension)
conn.execute("INSTALL parquet")
conn.execute("LOAD parquet")
conn.execute("""
    COPY person FROM '/path/to/person.parquet'
    WITH (FORMAT='parquet')
""", mode="insert")
```

## Error Handling

```python
from neug import NeuGError, NeuGQueryError, NeuGConnectionError

try:
    result = conn.execute("INVALID QUERY")
except NeuGQueryError as e:
    print(f"Query error: {e}")
except NeuGConnectionError as e:
    print(f"Connection error: {e}")
except NeuGError as e:
    print(f"General error: {e}")
```

## Configuration

### Database Config

```python
db = neug.Database(
    db_path="/path/to/db",
    mode="read_write",
    memory_level="high",
    max_thread_num=4
)
```

### Query Config

```python
result = conn.execute(
    query="MATCH ...",
    mode="read",
    parameters={"param": value},
    num_threads=2  # Override default thread count
)
```

## Complete Example

```python
import neug

# 1. Open database
db = neug.Database("/tmp/neug_db")

# 2. Create schema
conn = db.connect()
conn.execute("""
    CREATE VERTEX TYPE person IF NOT EXISTS {
        id INT64,
        fName STRING,
        lName STRING,
        age INT32,
        PRIMARY KEY (id)
    }
""", mode="schema")

conn.execute("""
    CREATE EDGE TYPE knows IF NOT EXISTS {
        since DATE
    }
    FROM person TO person WITH RELATION many_to_many
""", mode="schema")

# 3. Insert data
conn.execute("""
    CREATE (a:person {id: 1, fName: 'Alice', lName: 'Smith', age: 30}),
           (b:person {id: 2, fName: 'Bob', lName: 'Jones', age: 25}),
           (c:person {id: 3, fName: 'Charlie', lName: 'Brown', age: 35}),
           (a)-[:knows {since: date('2020-01-01')}]->(b),
           (b)-[:knows {since: date('2021-06-15')}]->(c)
""", mode="insert")

# 4. Query data
result = conn.execute("""
    MATCH (a:person)-[:knows]->(b:person)
    RETURN a.fName, b.fName
    ORDER BY a.fName
""")

print("Who knows whom:")
for row in result:
    print(f"  {row[0]} knows {row[1]}")

# 5. Update data
conn.execute("""
    MATCH (n:person {id: 1})
    SET n.age = 31
""", mode="update")

# 6. Clean up
conn.close()
db.close()
```

## References

- [Java Driver](./java-driver.md)
- [C++ API](./cpp-api.md)
- [Cypher Manual](./cypher-manual.md)