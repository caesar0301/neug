# Data Management Overview

This section documents NeuG's data loading and export capabilities.

## Documents

| Document | Description |
|----------|-------------|
| [data-loading.md](./data-loading.md) | Bulk loading (CSV, JSON, Parquet) |
| [data-export.md](./data-export.md) | Export formats and COPY TO syntax |
| [dataset-loader.md](./dataset-loader.md) | Built-in datasets (tinysnb, ldbc) |

## Supported Formats

| Format | Extension | File Types |
|--------|-----------|------------|
| CSV | Built-in | CSV, TSV, delimited text |
| JSON | `json` | JSON array, JSONL |
| Parquet | `parquet` | Apache Parquet |

## Loading Data

### CSV Format

```cypher
COPY person FROM '/path/to/person.csv' WITH (FORMAT='csv', DELIMITER=',');
```

### JSON Format

```cypher
COPY person FROM '/path/to/person.jsonl' WITH (FORMAT='json');
```

### Parquet Format

```cypher
COPY person FROM '/path/to/person.parquet' WITH (FORMAT='parquet');
```

## YAML Schema

```yaml
schema:
  vertex_types:
    - type_name: person
      type_id: 0
      primary_keys: [id]
      properties:
        - property_name: fName
          property_type: STRING
  edge_types:
    - type_name: knows
      type_id: 1
      vertex_type_pair_relations:
        - source_vertex: person
          destination_vertex: person
          relation: many_to_many
```

## Loading Configuration

| Option | Description |
|--------|-------------|
| `delimiter` | Field delimiter (default: `,`) |
| `header_row` | First row is header (default: `true`) |
| `batch_size` | Rows per batch |
| `parallelism` | Loading thread count |
| `null_values` | Values treated as NULL |

## Key Classes

| Class | Header | Purpose |
|-------|--------|---------|
| `LoadingConfig` | `storages/loader/loading_config.h` | Loading configuration |
| `CSVPropertyGraphLoader` | `storages/loader/csv_property_graph_loader.h` | CSV loader |
| `LoaderFactory` | `storages/loader/loader_factory.h` | Loader registry |

## Built-in Datasets

| Dataset | Description |
|---------|-------------|
| `tinysnb` | Small social network for testing |
| `ldbc` | LDBC Social Network Benchmark |