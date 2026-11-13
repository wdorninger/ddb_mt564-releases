# DuckDB-Wasm

*mt564 v0.1.1 · [README](../README.md)*

`mt564` is also built for [DuckDB-Wasm](https://github.com/duckdb/duckdb-wasm), so `read_mt564` can run
in a browser or in Node.js.

## Live demo

[ddb-gcloud.web.app](https://ddb-gcloud.web.app/) is a hosted DuckDB-Wasm page with `bberg` and `mt564`
preloaded, built from the [ddb_gcloud](https://github.com/wdorninger/ddb_gcloud) project. It is a working
reference for the setup described on this page.

## Target

| | |
|---|---|
| Platform | `wasm_eh`, the default DuckDB-Wasm bundle with native WebAssembly exceptions. `wasm_mvp` is not supported because the extension requires C++ exceptions. |
| Host | DuckDB-Wasm based on DuckDB 1.5.6 or newer. Tested with `@duckdb/duckdb-wasm@1.33.1-dev114.0` (DuckDB 2.0.0). |
| File | `mt564.duckdb_extension.wasm` |

## Loading

DuckDB-Wasm fetches extensions over HTTP(S), or reads them from a local path in Node.js. As with the native
builds, unsigned extensions must be allowed when the database is opened:

```js
await db.open({ allowUnsignedExtensions: true });
const conn = await db.connect();

// In a browser, the server must send CORS headers for the extension file.
await conn.query(`LOAD 'https://static.example.com/duckdb/mt564.duckdb_extension.wasm'`);

// Input files must be known to DuckDB-Wasm: registered buffers, file handles or HTTP URLs.
await db.registerFileBuffer('batch.txt', new Uint8Array(await file.arrayBuffer()));
const result = await conn.query(`
  SELECT ISIN, CAEV, CAOP, DFLT
  FROM read_mt564('batch.txt', level = 'CAOPTN')
`);
console.table(result.toArray().map(r => r.toJSON()));
```

## Differences from the native builds

- **No globs or directories.** Files registered with DuckDB-Wasm are not visible to the extension's file
  listing, so every input is treated as an explicit path and a glob pattern is an error. Pass a list, or build
  one with DuckDB's `glob()` function, which does see registered files:

  ```sql
  SET VARIABLE files = (SELECT list(file) FROM glob('swift/*.txt'));
  SELECT * FROM read_mt564(getvariable('files'));
  ```

- **Single-threaded.** The default `wasm_eh` bundle runs on one thread. `parallel` is accepted and does not
  change results.
- **Remote files** are read through DuckDB-Wasm's HTTP file system. In a browser, the server must allow
  CORS and HTTP range requests.

All other features behave as in the native builds: gzip, every `level`, type detection and NULL handling.
