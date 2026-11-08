# mt564 — DuckDB extension (binaries)

Prebuilt binaries of the `mt564` DuckDB extension, which adds `read_mt564` for reading
SWIFT FIN MT564 (Corporate Action Notification) message batches with SQL.

This repository contains **only the compiled extension**; the source code is not public.
Download the files from the [Releases](https://github.com/wdorninger/ddb_mt564-releases/releases) page.

## Files

| File | Platform |
|---|---|
| `mt564.linux_amd64.duckdb_extension` | Linux x86-64 (glibc ≥ 2.28) |
| `mt564.windows_amd64.duckdb_extension` | Windows x86-64 |
| `mt564.duckdb_extension.wasm` | DuckDB-Wasm (`wasm_eh`, browser / Node) |
| `SHA256SUMS` | checksums |

Works with **DuckDB ≥ 1.5.6, including 2.0.x** (built on DuckDB's stable C extension API).

## Usage

The extension is not signed by DuckDB, so unsigned extensions must be allowed:

```sh
duckdb -unsigned
```

```sql
LOAD '/path/to/mt564.linux_amd64.duckdb_extension';
SELECT ISIN, CAEV, CAOP FROM read_mt564('batch.txt', level='CAOPTN');
```

Python: `duckdb.connect(config={"allow_unsigned_extensions": "true"})`, then the same `LOAD`.

DuckDB-Wasm: open the database with `allowUnsignedExtensions: true` and
`LOAD 'https://<your host>/mt564.duckdb_extension.wasm'` (served with CORS). Globs are not available in the
Wasm build; pass explicit file paths or a list.

Keep the file name starting with `mt564.`: DuckDB derives the extension name from the part before the first dot.

## Third-party notices

See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
