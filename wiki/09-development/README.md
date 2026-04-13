# Development Guide Overview

This section provides guides for NeuG developers and contributors.

## Documents

| Document | Description |
|----------|-------------|
| [building-from-source.md](./building-from-source.md) | Build instructions |
| [code-style.md](./code-style.md) | Coding conventions |
| [testing-guide.md](./testing-guide.md) | Testing framework |
| [contributing.md](./contributing.md) | Contribution workflow |

## Building from Source

### Prerequisites

- CMake >= 3.15
- C++17 compiler (GCC 9+, Clang 10+, MSVC 2019+)
- Python 3.8+ (for bindings)

### Build Steps

```bash
git clone https://github.com/alibaba/neug.git
cd neug

mkdir build && cd build
cmake ..
make -j$(nproc)
```

### Build Python Wheel

```bash
pip install build
python -m build
```

## Project Structure

```
neug/
├── src/               # Source code
│   ├── compiler/      # Query compilation
│   ├── execution/     # Query execution
│   ├── storages/      # Storage layer
│   ├── transaction/   # Transaction management
│   ├── main/          # Main engine
│   └── server/        # Server mode
├── include/neug/      # Public headers
├── tools/             # Python/Java bindings
├── extension/         # Built-in extensions
├── tests/             # Test suites
├── doc/               # Documentation
└── proto/             # Protocol Buffers
```

## Code Style

- **Format**: ClangFormat with `.clang-format` config
- **Naming**: snake_case for functions/variables, CamelCase for classes
- **Comments**: Doxygen-style for public APIs

## Testing

```bash
# Run all tests
cd build
ctest --output-on-failure

# Run specific test
./tests/unittest/neug_test --gtest_filter=CSRTest*
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Write code and tests
4. Submit a pull request

See [CONTRIBUTING.md](../../CONTRIBUTING.md) for detailed guidelines.