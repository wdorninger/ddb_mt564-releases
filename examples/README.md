# Sample files

| File | Contents | Used in |
|---|---|---|
| [`notifications.txt`](notifications.txt) | Three MT564 notifications: cash dividend (`DVCA`), dividend with scrip option (`DVOP`), merger (`MRGR`) | [README](../README.md#example), [read_mt564](../docs/mt564.md) |

To run the documented queries, start DuckDB in the repository root:

```sh
duckdb -unsigned -c "LOAD 'mt564.linux_amd64.duckdb_extension'; SELECT * FROM read_mt564('examples/notifications.txt');"
```

## Disclaimer

This file contains fictitious test data created solely for documentation purposes. Messages, BICs,
references, ISINs, security names, account numbers, rates, amounts and dates are invented and do not refer to
any real institution, security, account or corporate action. Any resemblance to actual entities or data is
coincidental. No production SWIFT traffic is included.
