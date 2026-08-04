# Memory-spike reproduction CSVs (holdings import)

Supporting files for
[`docs/architecture/memory-spike-holdings-import-rca.md`](../../architecture/memory-spike-holdings-import-rca.md).
Byte-identical 51-ticker holdings CSVs differing **only in the date field**, forming a
controlled experiment for the unbounded-valuation-window bug.

| File | Date field | Measured result |
|---|---|---|
| `holdings-control-2024.csv` | all rows `2024-07-20` | < 200 MB, completes in seconds |
| `holdings-dose-1924.csv` | all rows `1924-07-20` | intermediate (~37k-day window), not run |
| `holdings-repro-oom.csv` | one row (DIS) `0224-07-20` | 2 GB spike, settles at 600 MB |
| `holdings-repro-oom-v2.csv` | all rows `0224-07-20` | **runaway allocation → OS kills the app** |

⚠️ **`holdings-repro-oom-v2.csv` intentionally OOMs the app.** Import only into a disposable
test profile/account, with a DB backup. The poisoned rows persist — clean up afterwards:

```sql
DELETE FROM holdings_snapshots WHERE snapshot_date < '1990-01-01';
```

then trigger a recalculation. These files become obsolete once fixes F1/F2 land (the import
should then reject the bad dates / clamp the window); at that point they serve as regression
inputs.
