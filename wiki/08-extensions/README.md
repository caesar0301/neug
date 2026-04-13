# Extensions Overview

This section documents NeuG's extension system.

## Documents

| Document | Description |
|----------|-------------|
| [extension-development.md](./extension-development.md) | How to create extensions |
| [json-extension.md](./json-extension.md) | JSON format support |
| [parquet-extension.md](./parquet-extension.md) | Parquet format support |

## Extension Architecture

```
Extension (shared library)
├── Init()              # Entry point - register functions
├── Name()              # Extension identifier
└── Registered functions
    ├── Table functions (scan)
    ├── Copy functions (export)
    └── File systems (S3, etc.)
```

## Built-in Extensions

| Extension | Purpose |
|-----------|---------|
| `json` | JSON/JSONL file reading and export |
| `parquet` | Apache Parquet file reading |

## Loading Extensions

```python
import neug

db = neug.Database("/path/to/db")
conn = db.connect()

# Install extension (download if needed)
conn.execute("INSTALL json")

# Load extension
conn.execute("LOAD json")

# Use extension
conn.execute("COPY person FROM '/data/person.jsonl' WITH (FORMAT='json')")
```

## Extension API

```cpp
// extension/my_extension/src/my_extension.cpp
#include "neug/compiler/extension/extension_api.h"

extern "C" {

void Init(neug::ExtensionAPI* api) {
    // Register table function
    api->registerFunction<neug::MyTableFunction>();

    // Register copy function
    api->registerFunction<neug::MyExportFunction>();

    // Register extension info
    neug::ExtensionInfo info;
    info.name = "my_extension";
    info.version = "1.0.0";
    api->registerExtension(info);
}

const char* Name() {
    return "my_extension";
}

}  // extern "C"
```

## Key Classes

| Class | Header | Purpose |
|-------|--------|---------|
| `ExtensionAPI` | `extension/extension_api.h` | Registration API |
| `Extension` | `extension/extension.h` | Extension loading utilities |
| `ExtensionLibLoader` | `extension/extension.h` | Dynamic library loading |

## Extension Directory Structure

```
extension/my_extension/
├── CMakeLists.txt
├── include/
│   └── my_function.h
└── src/
    ├── my_extension.cpp    # Init(), Name()
    └── my_function.cpp     # Function implementations
```