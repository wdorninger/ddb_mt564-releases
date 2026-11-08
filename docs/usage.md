# Using `read_mt564`

*Documentation for `mt564` v0.1.1. [Back to the README](../README.md).*

This page covers inputs, types, NULLs and scanning behaviour. The MT564 structure, the `level=` grains and
the output columns are described in [mt564.md](mt564.md).

## Loading

```sql
LOAD '/path/to/mt564.linux_amd64.duckdb_extension';   -- needs allow_unsigned_extensions / duckdb -unsigned
```

The extension works with DuckDB 1.5.6 and newer, including 2.0.x. See
[DuckDB version compatibility](../README.md#duckdb-version-compatibility). For DuckDB-Wasm (browser/Node) see
[wasm.md](wasm.md): there, inputs must be explicit paths or lists, because globs and directories are not
available.

## Inputs

The first argument is the input. It can be any of the following:

| Form | Example | Notes |
|---|---|---|
| single file | `'swift/batch.txt'` | |
| gzip file | `'swift/batch.txt.gz'` | decoded when the name ends in `.gz` **and** the file starts with the gzip magic bytes; a `.gz`-named plain file is read as text; multi-member gzip is supported |
| directory | `'swift/2026/'` | every file below it, recursively |
| glob | `'swift/*/*.txt'`, `'swift/**/*.fin'` | `*`, `?`, `[...]`, and `**` for any depth. Local paths only |
| list | `['a.txt', 'b.txt', 'archive/*.txt']` | each entry may itself be a glob or directory; the list order is kept, and duplicate files are dropped |
| remote | `'s3://bucket/x.txt'`, `'https://host/x.txt.gz'` | needs `httpfs`; explicit paths (or lists of them), not globs |

Within one glob or directory, files are sorted by path. A glob that matches nothing is an error
(`No files found that match the pattern ...`). A missing explicit path fails with DuckDB's own
`Cannot open file` error.

### Remote storage (S3, GCS, Azure via httpfs, HTTP)

Every byte is read through the file system of the DuckDB instance that loaded the extension, so remote
storage uses the same secrets, credential chains, proxies and caching as the rest of DuckDB:

```sql
INSTALL httpfs; LOAD httpfs;
CREATE SECRET swift (TYPE s3, PROVIDER credential_chain, REGION 'eu-west-1');

SELECT * FROM read_mt564(['s3://my-bucket/swift/a.txt', 's3://my-bucket/swift/b.txt'], level='CAOPTN');
```

Remote files are read in 8 MB blocks (local files in 256 KB blocks). Globs are not expanded on remote paths,
because DuckDB's C extension API has no glob function. Build the list in SQL first instead:

```sql
SET VARIABLE files = (SELECT list(file) FROM glob('s3://my-bucket/swift/*.txt.gz'));
SELECT * FROM read_mt564(getvariable('files'));
```

### Replacement scan

A quoted path ending in `.mt564` (case-insensitive, optionally followed by `.gz`) in `FROM` calls
`read_mt564` with default options:

```sql
SELECT * FROM 'swift/batch.mt564';
```

SWIFT exports often use `.txt` or `.fin`; call `read_mt564` explicitly for those, and to pass options.

## Extra output columns

After the format's columns come:

- `filename VARCHAR`, when `filename=true`: the path exactly as it was opened.
- One `VARCHAR` column per hive partition key, when `hive_partitioning=true`. Path segments such as
  `year=2026/month=11` become columns `year` and `month`. Keys are matched case-insensitively across files,
  and a key missing from a file's path is `NULL` for that file's rows.

## Types

Qualifier columns are typed from the first `sample_size` messages (default 2000). The first type that every
sampled, non-empty, non-NULL-marker value fits is chosen:

```
DATE     (exactly 8 digits forming a valid YYYYMMDD date)
BOOLEAN  (true/false/t/f/1/0, any case)
  -> BIGINT  (integers that fit in 64 bits)
    -> DOUBLE  (decimal or exponent notation)
      -> VARCHAR
```

Zero-padded values such as option number `CAON` `001` are numbers, because SWIFT zero-pads genuine numeric
fields. `all_varchar=true` (or `auto_detect=false`) makes every discovered column `VARCHAR`. A value that
does not parse as its column's type becomes `NULL`. Number parsing does not depend on the process locale.

## NULL handling

| Option | Effect |
|---|---|
| `nullstr='UKWN'` or `nullstr=['UKWN', 'NONE']` | trimmed values equal to a marker become `NULL`; markers are also ignored by type detection |
| `empty_as_null=true` | empty `VARCHAR` values become `NULL` (an empty value in a non-`VARCHAR` column is always `NULL`) |

Both apply to the discovered qualifier columns, not to the fixed `FUNCTION`, `ISIN`, `SECURITY_ALT_ID` and
`SECURITY_DESC` columns.

## Multiple files and parallelism

- Files are scanned in parallel, one file per thread, with up to one thread per core. Within a file, rows
  keep file order. Across files, the order is unspecified unless you `ORDER BY`. `parallel=false` reads the
  files one after another in input order.
- A single file is always read by one thread. To use more cores on one very large batch, split it.
- Column discovery samples messages across all files until `sample_size` is reached. A qualifier present in
  one file but absent in another is `NULL` where it is missing.

## Performance tips

- **Select only the columns you need.** Projection pushdown means unselected columns are never parsed; the
  `OTHER_FIELDS` map in particular costs nothing when not selected.
- **Choose the coarsest `level` that answers the question.** `MESSAGE` with the `NUM_*` counts is enough to
  find events; drill into `CAOPTN`/`CASHMOVE`/`SECMOVE` only when you need those rows.
- **gzip costs CPU.** Uncompressed local files are fastest; for remote storage gzip usually wins.

## Errors

| Situation | Behaviour |
|---|---|
| truncated or corrupt gzip | error (`unexpected end of file (truncated archive)` / zlib message) |
| glob with no matches | error |
| invalid `level=` or `nullstr` type | bind-time error naming the valid values |
| `store_rejects=true` / `rejects_scan` | error: reject tables are not supported |
