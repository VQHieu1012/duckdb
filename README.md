# DuckDB — Bounded-Memory CSV Reject Handling

A custom DuckDB fork that addresses out-of-memory (OOM) failures when processing large CSV files containing millions of malformed records.

Based on DuckDB `v2.0.0`.

## Problem Statement

When reading large CSV files with `ignore_errors = true` and `store_rejects = true`, DuckDB accumulates CSV error objects in memory before populating the reject table.

This can cause excessive heap memory consumption and eventually lead to out-of-memory failures, even when `rejects_limit` is configured.

Common symptoms include:

- Continuously increasing process memory when reading malformed CSV files.
- OOM failures when enabling `store_rejects`.
- `rejects_limit` not preventing excessive memory consumption during scanning.
- Large numbers of redundant `MISSING COLUMNS` error entries.

## Root Cause

The original CSV error handler accumulates ignored errors in a `vector<CSVError>` during scanning.

```cpp
if (store_ignored_errors) {
    errors.push_back(csv_error);
}
```

The reject limit is applied later when populating the reject table, rather than when collecting errors.

These error objects are allocated on the heap and are not automatically managed by DuckDB's spill-to-disk mechanism.

## Solution

This fork introduces a bounded-buffer streaming reject writer that writes malformed CSV records directly to a JSONL file during scanning.

Instead of accumulating all rejected records in memory, errors are serialized and written incrementally using a configurable byte buffer.

The implementation also eliminates redundant `MISSING COLUMNS` errors by generating a single missing-column error per malformed row.

### Features

- Streaming CSV reject output to JSONL.
- Configurable reject buffer size.
- Support for unlimited reject entries without retaining the entire ignored-error collection in memory.
- Reduced error amplification for missing columns.
- Preservation of rejected record content, byte positions, error types, and error messages.
- Early cap on cached errors in traditional reject-table mode.
- No additional external dependencies for the reject writer.

## Usage

Two custom options are available in the CSV reader:

| Option | Type | Default | Description |
|---|---|---|---|
| `rejects_path` | VARCHAR | Unset | Output path for rejected records |
| `rejects_buffer_size` | BIGINT | 8388608 | Reject writer buffer size in bytes |

### SQL Example

```sql
SELECT *
FROM read_csv(
    'input.csv',
    auto_detect = false,
    delim = ',',
    header = true,
    columns = {
        'id': 'BIGINT',
        'name': 'VARCHAR',
        'value': 'BIGINT'
    },
    ignore_errors = true,
    store_rejects = true,
    rejects_limit = 0,
    rejects_path = '/data/rejects.jsonl',
    rejects_buffer_size = 8388608
);
```

Setting `rejects_limit = 0` allows unlimited reject entries in streaming mode.

The query returns valid records while malformed records are written to the specified JSONL file.

### Reject Output

For the following input:

```csv
id,name,value
1,Alice,100
2,Bob,200,EXTRA
3,Charlie,300
4,David,abc
5,Eva
6,Frank,600
```

The valid records are:

```text
1,Alice,100
3,Charlie,300
6,Frank,600
```

The reject file contains JSONL entries such as:

```json
{"line_byte_position":27,"byte_position":36,"error_type":"TOO MANY COLUMNS","value":"2,Bob,200,EXTRA","error_message":"Expected Number of Columns: 3 Found: 4"}
{"line_byte_position":57,"byte_position":65,"error_type":"CAST","value":"4,David,abc","error_message":"Error when converting column \"value\". Could not convert string \"abc\" to 'BIGINT'"}
{"line_byte_position":69,"byte_position":74,"error_type":"MISSING COLUMNS","value":"5,Eva","error_message":"Expected Number of Columns: 3 Found: 2"}
```

## Benchmark

The implementation was tested on a 5.2 GiB CSV file containing more than 110 million physical data lines.

### Dataset

| Metric | Count |
|---|---:|
| Total data lines | 110,513,395 |
| Valid records | 84,200,000 |
| Missing-column records | 21,051,521 |
| Extra-column records | 5,261,874 |
| Total malformed records | 26,313,395 |

### Results

| Metric | Streaming V1 | Streaming V2 |
|---|---:|---:|
| Execution time | 128.791 s | 67.355 s |
| Valid records | 84,200,000 | 84,200,000 |
| Unique rejected positions | 26,313,395 | 26,313,395 |
| Reject entries | 52,629,160 | 26,313,395 |
| Reject file size | 9.1 GiB | 4.7 GiB |
| OOM | No | No |

The optimized implementation achieved:

- **1.91x execution speedup.**
- **47.7% reduction in execution time.**
- **Approximately 50% fewer reject entries.**
- **Approximately 48% smaller reject output.**
- **No loss of rejected record positions compared with V1.**

The rejected record positions were verified by sorting distinct byte offsets and comparing SHA-256 checksums between both implementations.

The original OOM issue was mitigated on the tested workload by replacing unbounded ignored-error accumulation with incremental file writes.

These benchmark results are workload-specific and are not general performance guarantees.

## Build From Source

```bash
git clone --branch feature/bounded-csv-rejects \
    https://github.com/VQHieu1012/duckdb.git

cd duckdb

GEN=ninja CMAKE_BUILD_PARALLEL_LEVEL=2 make release
```

The resulting DuckDB CLI binary can be found at:

```text
build/release/duckdb
```

For upstream build requirements, refer to the official [DuckDB build documentation](https://duckdb.org/docs/current/dev/building/overview).

## Important Notes

- Write reject files to a disk-backed filesystem rather than a memory-backed filesystem such as `tmpfs` when processing large datasets.
- The reject buffer size limits buffered output data, not the total memory consumption of DuckDB.
- The output file must not already exist.
- Streaming reject output currently supports single-file CSV scans.
- Interrupted queries may leave incomplete JSONL output.
- Traditional reject-table mode has a hard-coded cap of 1,000 cached ignored errors.
- This fork does not introduce transactional guarantees for reject output.
- The custom implementation applies to CSV scanning only.

## Project Status

This fork focuses on solving excessive memory consumption caused by CSV reject collection in large-scale ingestion workloads.

It is not intended to replace DuckDB's general CSV ingestion architecture or provide a complete dead-letter queue system.

Full compatibility with all upstream regression tests has not been established.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for detailed implementation changes, benchmark results, and commit history.

## Upstream

This project is a fork of the official [DuckDB repository](https://github.com/duckdb/duckdb).

All original DuckDB licensing terms are preserved.

---

## Original DuckDB README
<div align="center">
  <picture>
    <source media="(prefers-color-scheme: light)" srcset="logo/DuckDB_Logo-horizontal.svg">
    <source media="(prefers-color-scheme: dark)" srcset="logo/DuckDB_Logo-horizontal-dark-mode.svg">
    <img alt="DuckDB logo" src="logo/DuckDB_Logo-horizontal.svg" height="100">
  </picture>
</div>
<br>

<p align="center">
  <a href="https://github.com/duckdb/duckdb/actions"><img src="https://github.com/duckdb/duckdb/actions/workflows/Main.yml/badge.svg?branch=main" alt="Github Actions Badge"></a>
  <a href="https://discord.gg/tcvwpjfnZx"><img src="https://shields.io/discord/909674491309850675" alt="discord" /></a>
  <a href="https://github.com/duckdb/duckdb/releases/"><img src="https://img.shields.io/github/v/release/duckdb/duckdb?color=brightgreen&display_name=tag&logo=duckdb&logoColor=white" alt="Latest Release"></a>
</p>

## DuckDB

DuckDB is a high-performance analytical database system. It is designed to be fast, reliable, portable, and easy to use. DuckDB provides a rich SQL dialect with support far beyond basic SQL. DuckDB supports arbitrary and nested correlated subqueries, window functions, collations, complex types (arrays, structs, maps), and [several extensions designed to make SQL easier to use](https://duckdb.org/docs/current/sql/dialect/friendly_sql.html).

DuckDB is available as a [standalone CLI application](https://duckdb.org/docs/current/clients/cli/overview) and has clients for [Python](https://duckdb.org/docs/current/clients/python/overview), [R](https://duckdb.org/docs/current/clients/r), [Java](https://duckdb.org/docs/current/clients/java), [Wasm](https://duckdb.org/docs/current/clients/wasm/overview), etc., with deep integrations with packages such as [pandas](https://duckdb.org/docs/guides/python/sql_on_pandas) and [dplyr](https://duckdb.org/docs/current/clients/r#duckplyr-dplyr-api).

For more information on using DuckDB, please refer to the [DuckDB documentation](https://duckdb.org/docs/current/).

## Installation

If you want to install DuckDB, please see [our installation page](https://duckdb.org/docs/installation/) for instructions.

## Data Import

For CSV files and Parquet files, data import is as simple as referencing the file in the FROM clause:

```sql
SELECT * FROM 'myfile.csv';
SELECT * FROM 'myfile.parquet';
```

Refer to our [Data Import](https://duckdb.org/docs/current/data/overview) section for more information.

## SQL Reference

The documentation contains a [SQL introduction and reference](https://duckdb.org/docs/current/sql/introduction).

## Development

For development, DuckDB requires [CMake](https://cmake.org), Python 3 and a `C++17` compliant compiler. In the root directory, run `make` to compile the sources. For development, use `make debug` to build a non-optimized debug version. You should run `make unit` and `make allunit` to verify that your version works properly after making changes. To test performance, you can run `BUILD_BENCHMARK=1 BUILD_TPCH=1 make` and then perform several standard benchmarks from the root directory by executing `./build/release/benchmark/benchmark_runner`. The details of benchmarks are in our [Benchmark Guide](benchmark/README.md).

Please also refer to our [Build Guide](https://duckdb.org/docs/current/dev/building/overview) and [Contribution Guide](CONTRIBUTING.md).

## Support

See the [Support Options](https://ducklabs.com/support/) page and the dedicated [`endoflife.date`](https://endoflife.date/duckdb) page.
