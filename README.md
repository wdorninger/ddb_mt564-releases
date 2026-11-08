# mt564 — DuckDB extension (binaries)

Prebuilt binaries of the `mt564` DuckDB extension, which adds `read_mt564` for reading
SWIFT FIN MT564 (Corporate Action Notification) message batches with SQL.

This repository contains **only the compiled extension**; the source code is not public.
Current version: **v0.1.1**. Download the files from the [Releases](https://github.com/wdorninger/ddb_mt564-releases/releases) page.

## Files

| File | Platform |
|---|---|
| `mt564.linux_amd64.duckdb_extension` | Linux x86-64 (glibc ≥ 2.28) |
| `mt564.osx_arm64.duckdb_extension` | macOS Apple Silicon (macOS ≥ 11) |
| `mt564.osx_amd64.duckdb_extension` | macOS Intel (macOS ≥ 11) |
| `mt564.windows_amd64.duckdb_extension` | Windows x86-64 |
| `mt564.duckdb_extension.wasm` | DuckDB-Wasm (`wasm_eh`, browser / Node) |
| `SHA256SUMS` | checksums |

## DuckDB version compatibility

Works with **DuckDB ≥ 1.5.6, including 2.0.x**. The extension is built on DuckDB's stable C extension API (version
v1.5.6), so the same binary loads into every newer DuckDB release without rebuilding. Remote storage (httpfs/S3),
secrets and caching come from the DuckDB that loads it.

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

macOS: a downloaded file is quarantined by Gatekeeper; clear the flag before loading it with
`xattr -d com.apple.quarantine mt564.osx_arm64.duckdb_extension`.

Keep the file name starting with `mt564.`: DuckDB derives the extension name from the part before the first dot.

## Documentation

Documentation for v0.1.1:

| Page | Contents |
|---|---|
| [docs/mt564.md](docs/mt564.md) | `read_mt564`: message structure, `level=` grains, columns, worked examples, options |
| [docs/usage.md](docs/usage.md) | Inputs, globs, remote files, replacement scan, types, NULLs, parallelism, errors |
| [docs/wasm.md](docs/wasm.md) | Using the DuckDB-Wasm build in a browser or Node |
| [examples/deep_dive.txt](examples/deep_dive.txt) | the (anonymized) example messages used in docs/mt564.md |

## Third-party notices

See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
