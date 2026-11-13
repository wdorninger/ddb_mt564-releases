# Shared behaviour

*mt564 v0.1.1 · [README](../README.md)*

This page covers inputs, types, NULL handling and scanning behaviour of [`read_mt564`](mt564.md). The message
structure, the `level` option and the output columns are described in [mt564.md](mt564.md).

## Loading

```sql
LOAD '/path/to/mt564.linux_amd64.duckdb_extension';
```

The database must allow unsigned extensions (`duckdb -unsigned`, or `allow_unsigned_extensions` in the
client configuration). For Java see [jdbc.md](jdbc.md), for DuckDB-Wasm see [wasm.md](wasm.md).

## Inputs

The first argument selects the input:

| Form | Example | Notes |
|---|---|---|
| File | `'swift/batch.txt'` | |
| gzip file | `'swift/batch.txt.gz'` | Decompressed when the name ends in `.gz` and the content starts with the gzip signature. Multi-member archives are supported. |
| Directory | `'swift/2026/'` | All files below the directory, recursively |
| Glob | `'swift/*/*.txt'`, `'swift/**/*.fin'` | `*`, `?`, `[...]`, and `**` for any depth. Local paths only. |
| List | `['a.txt', 'b.txt', 'archive/*.txt']` | Entries may be globs or directories. Order is kept and duplicates are removed. |
| Remote | `'s3://bucket/x.txt'`, `'https://host/x.txt.gz'` | Requires `httpfs`. Explicit paths or lists only. |

Files matched by one glob or directory are processed in path order. A glob without matches raises
`No files found that match the pattern ...`; a missing file raises DuckDB's `Cannot open file` error.

Within a file, each message must start on a new line.

### Remote storage

All reads go through the file system of the DuckDB instance that loaded the extension. Secrets, credential
chains, proxies and the HTTP cache of that instance apply unchanged.

```sql
INSTALL httpfs;
LOAD httpfs;
CREATE SECRET swift_archive (TYPE s3, PROVIDER credential_chain, REGION 'eu-west-1');

SELECT * FROM read_mt564(['s3://example-bucket/swift/a.txt', 's3://example-bucket/swift/b.txt'],
                         level = 'CAOPTN');
```

Remote files are read in 8 MB blocks (local files in 256 KB blocks). Globs are not expanded on remote paths
because DuckDB's C extension API does not expose glob expansion. Build the file list with DuckDB's `glob()`
function instead:

```sql
SET VARIABLE files = (SELECT list(file) FROM glob('s3://example-bucket/swift/*.txt.gz'));
SELECT * FROM read_mt564(getvariable('files'));
```

### Replacement scan

A quoted path ending in `.mt564` (case-insensitive, optionally followed by `.gz`) in the `FROM` clause calls
`read_mt564` with default options:

```sql
SELECT * FROM 'swift/batch.mt564';
```

Exports often use `.txt` or `.fin`. Call `read_mt564` directly for those files, and to pass options.

## Additional columns

- `filename VARCHAR` with `filename = true`: the path as it was opened.
- One `VARCHAR` column per hive partition key with `hive_partitioning = true`. A path such as
  `year=2026/month=01/batch.txt` adds the columns `year` and `month`. Keys are matched case-insensitively
  across files; a key missing from a path is `NULL` for that file.

## Type detection

Qualifier columns are typed from the first `sample_size` messages (default 2000). The first type that all
sampled values fit is used. Empty values and NULL markers are ignored.

| Order | Type | Accepted values |
|---:|---|---|
| 1 | `DATE` | Exactly eight digits forming a valid `YYYYMMDD` date |
| 2 | `BOOLEAN` | `true`, `false`, `t`, `f`, `1`, `0` (any case) |
| 3 | `BIGINT` | Integers within 64 bits |
| 4 | `DOUBLE` | Decimal or exponent notation |
| 5 | `VARCHAR` | Anything else |

Zero-padded values such as the option number `CAON` `001` are detected as numbers, because SWIFT zero-pads
numeric fields. `all_varchar = true` or `auto_detect = false` makes all qualifier columns `VARCHAR`. A value
that cannot be converted to its column type becomes `NULL`. Number parsing does not depend on the process
locale.

## NULL handling

| Option | Effect |
|---|---|
| `nullstr = 'UKWN'` or `nullstr = ['UKWN', 'NONE']` | Values equal to a marker after trimming become `NULL`. Markers are ignored by type detection. |
| `empty_as_null = true` | Empty `VARCHAR` values become `NULL`. Empty values in other column types are always `NULL`. |

Both options apply to qualifier columns, not to `FUNCTION`, `ISIN`, `SECURITY_ALT_ID` and `SECURITY_DESC`.

## Parallelism

- Files are scanned in parallel, one file per thread, up to one thread per core. Rows of one file keep file
  order. Across files, the order is not defined unless the query has an `ORDER BY`. `parallel = false` reads
  files one after another in input order.
- A single file is always read by one thread. Split very large batches to use more cores.
- Column discovery samples messages across all files until `sample_size` is reached. A qualifier present in
  one file but not in another is `NULL` where it is missing.

## Performance

- Select only the columns you need. Unselected columns are not parsed; this matters most for `OTHER_FIELDS`.
- Use the coarsest `level` that answers the question. `MESSAGE` with its `NUM_*` counts is enough to find
  events; use `CAOPTN`, `CASHMOVE` or `SECMOVE` when their fields are needed.
- Uncompressed local files are fastest. For remote storage, gzip usually wins because less data is
  transferred.

## Errors

| Situation | Behaviour |
|---|---|
| Truncated or corrupt gzip | Error (`unexpected end of file (truncated archive)` or the zlib message) |
| Glob without matches | Error |
| Invalid `level` or `nullstr` type | Error when the query is bound, naming the valid values |
| `store_rejects = true` or `rejects_scan` | Error; reject tables are not supported |
