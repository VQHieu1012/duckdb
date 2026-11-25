# Custom DuckDB Changelog

This document tracks changes introduced in this fork of DuckDB.

## v2.0.0-custom.1 — 2026-11-25

### Upstream Base

- DuckDB version: `v2.0.0`
- Base commit: `d54cbbe3df1`
- Custom engine commit: `9d56720cc0c`
- Branch: `feature/bounded-csv-rejects`

### Motivation

When processing large CSV files containing millions of malformed records, DuckDB's CSV reject collection can accumulate error objects in memory before populating the reject table.

This behavior can lead to excessive memory consumption and out-of-memory failures, even when `rejects_limit` is configured.

This fork introduces bounded-memory reject handling and streaming reject output to support large-scale data ingestion pipelines.

### Added

- Introduced the `rejects_path` CSV reader option to write rejected records directly to a JSONL file.
- Introduced the `rejects_buffer_size` option to configure the reject writer buffer size in bytes.
- Implemented `CSVRejectFileWriter` with buffered, incremental file writes.
- Added JSON serialization for rejected CSV records.
- Preserved support for `rejects_limit` in streaming reject mode, where `0` means unlimited.
- Added validation for reject output configuration.
- Restricted streaming reject output to single-file CSV scans.

### Fixed

- Mitigated unbounded accumulation of ignored CSV errors.
- Applied an early limit of 1,000 cached ignored errors in traditional reject-table mode.
- Bypassed the in-memory ignored-error collection when streaming reject output is enabled.

### Performance

- Eliminated redundant `MISSING COLUMNS` error objects.
- Replaced per-missing-column error generation with a single error per malformed row.
- Preserved the actual detected column count in the error message.
- Reduced unnecessary error allocations, serialization operations, mutex acquisitions, and disk writes.

### Configuration

| Option | Type | Default | Description |
|---|---|---|---|
| `rejects_path` | VARCHAR | Unset | Path to the reject JSONL file |
| `rejects_buffer_size` | BIGINT | 8388608 | Reject writer buffer size in bytes |
| `rejects_limit` | BIGINT | 0 | Maximum number of stored reject entries; zero means unlimited in streaming mode |
| `store_rejects` | BOOLEAN | false | Enables reject collection |
| `ignore_errors` | BOOLEAN | false | Allows scanning to continue after supported record errors |

### Usage

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

### Reject Output Format

Each reject entry is written as a JSON object followed by a newline.

```json
{
    "line_byte_position": 62,
    "byte_position": 67,
    "error_type": "MISSING COLUMNS",
    "value": "2,Bob",
    "error_message": "Expected Number of Columns: 5 Found: 2"
}
```

### Benchmark

The implementation was tested using a 5.2 GiB CSV file containing 110,513,395 physical data lines.

| Metric | Streaming V1 | Streaming V2 |
|---|---:|---:|
| Valid records | 84,200,000 | 84,200,000 |
| Unique rejected positions | 26,313,395 | 26,313,395 |
| Reject entries | 52,629,160 | 26,313,395 |
| Execution time | 128.791 s | 67.355 s |
| Reject file size | 9.1 GiB | 4.7 GiB |
| Out-of-memory failure | No | No |

Benchmark results:

- Approximately 1.91x execution speedup.
- Approximately 47.7% reduction in execution time.
- Approximately 50% reduction in reject entries.
- Approximately 48% reduction in reject file size.
- Identical sets of rejected byte offsets between V1 and V2, verified using SHA-256 checksums.

These results represent the tested workload and are not general performance guarantees.

### Known Limitations

- Streaming reject output currently supports single-file CSV scans only.
- The output file must not already exist.
- Interrupted scans may produce incomplete JSONL output.
- Buffered reject entries may be lost if the process terminates unexpectedly.
- Reject file writes do not provide transactional or durability guarantees.
- Reject output is uncompressed.
- Traditional reject-table mode retains a hard-coded cap of 1,000 cached ignored errors.
- The configured buffer size limits buffered reject data, not total process memory.
- Writing reject files to a memory-backed filesystem such as `tmpfs` may increase system memory consumption.
- Changes apply to CSV scanning only.
- Full upstream regression-suite compatibility has not been established.

### Implementation History

| Commit | Description |
|---|---|
| `02283a066f8` | Cap cached ignored errors to mitigate OOM |
| `b5c9cd8f80f` | Add `rejects_path` and `rejects_buffer_size` options |
| `e391997f133` | Implement bounded-buffer streaming reject file writer |
| `9d56720cc0c` | Emit a single missing-columns error per malformed row |

### Intended Use

This fork is designed for large-scale CSV ingestion pipelines requiring malformed record collection without retaining the entire reject collection in memory.

The primary use case is File Importer V2, where valid records are forwarded to Kafka data topics and rejected records are subsequently read from JSONL files and forwarded to dead-letter queues.
