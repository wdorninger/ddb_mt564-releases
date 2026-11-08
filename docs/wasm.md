# DuckDB-Wasm

*Documentation for `mt564` v0.1.0. [Back to the README](../README.md).*

The extension is also built for [DuckDB-Wasm](https://github.com/duckdb/duckdb-wasm), so
`read_mt564` can run in a browser or in Node.

## Target

| | |
|---|---|
| DuckDB platform | `wasm_eh`: DuckDB-Wasm's default bundle, with native wasm exceptions. The extension relies on C++ exceptions, so `wasm_mvp` is not offered |
| Host | DuckDB-Wasm built on DuckDB ≥ 1.5.6. Tested target: npm `@duckdb/duckdb-wasm@1.33.1-dev114.0`, which is DuckDB v2.0.0 |

## Load

DuckDB-Wasm fetches extensions over HTTP(S), or reads them from a local path in Node. Unsigned extensions
must be allowed when the database is opened:

```js
await db.open({ allowUnsignedExtensions: true });
const con = await db.connect();
await con.query(`LOAD 'https://example.com/ext/mt564.duckdb_extension.wasm'`);   // browser: needs CORS

// files must be known to DuckDB-Wasm: registered buffers/handles or HTTP URLs
await db.registerFileBuffer('batch.txt', new Uint8Array(await file.arrayBuffer()));
await con.query(`SELECT * FROM read_mt564('batch.txt', level='CAOPTN')`);
```

## Differences from the native build

- **No globs or directories.** DuckDB-Wasm's files (registered buffers, file handles, HTTP) are not visible
  to `std::filesystem`, and the C API cannot list them. Every input is treated as an explicit file path, and a
  glob pattern is an error. Pass a list, or build one in SQL with DuckDB's `glob()`, which does see
  registered files:
  ```sql
  SET VARIABLE files = (SELECT list(file) FROM glob('swift/*.txt'));
  SELECT * FROM read_mt564(getvariable('files'));
  ```
- **Single-threaded** in the default `wasm_eh` bundle. `parallel` is accepted, and results are identical.
- Remote files work through DuckDB-Wasm's HTTP file system. In a browser the server must allow CORS and
  range requests.
- Everything else is the same code: every `level=`, gzip, type detection, the `.mt564` replacement scan.

