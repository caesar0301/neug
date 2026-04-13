# Client APIs Overview

This section documents NeuG's client-facing APIs.

## Documents

| Document | Description |
|----------|-------------|
| [python-api.md](./python-api.md) | Python binding reference |
| [java-driver.md](./java-driver.md) | Java driver reference |
| [cpp-api.md](./cpp-api.md) | C++ API reference |
| [cypher-manual.md](./cypher-manual.md) | Supported Cypher syntax |

## Quick Examples

### Python

```python
import neug

# Embedded mode
db = neug.Database("/path/to/db")
db.load_builtin_dataset("tinysnb")

conn = db.connect()
result = conn.execute("""
    MATCH (a:person)-[:knows]->(b:person)
    RETURN a.fName, b.fName
""")

for row in result:
    print(f"{row[0]} knows {row[1]}")

# Service mode
conn.close()
db.serve(port=8080)
```

### Java

```java
import com.alibaba.neug.driver.*;

Driver driver = GraphDatabase.driver("neug://localhost:8080");
Session session = driver.session();

ResultSet result = session.run(
    "MATCH (n:person) WHERE n.id = $id RETURN n.fName",
    Values.parameters("id", 1),
    AccessMode.READ
);

while (result.next()) {
    System.out.println(result.getString(0));
}

session.close();
driver.close();
```

## Access Modes

| Mode | Python | Java | Description |
|------|--------|------|-------------|
| Read | `"r"` or `"read"` | `AccessMode.READ` | Read-only queries |
| Insert | `"i"` or `"insert"` | `AccessMode.INSERT` | Bulk inserts |
| Update | `"u"` or `"update"` | `AccessMode.UPDATE` | Read/write queries |
| Schema | `"s"` or `"schema"` | `AccessMode.SCHEMA` | DDL operations |

## Key Classes

### Python (`tools/python_bind/`)

| Class | File | Purpose |
|-------|------|---------|
| `Database` | `neug/database.py` | Database management |
| `Connection` | `neug/connection.py` | Query execution |
| `Session` | `neug/session.py` | Remote session (TP mode) |
| `QueryResult` | `neug/query_result.py` | Result iteration |

### Java (`tools/java_driver/`)

| Class | Purpose |
|-------|---------|
| `GraphDatabase` | Driver factory |
| `Driver` | Connection management |
| `Session` | Query execution |
| `ResultSet` | Result iteration |