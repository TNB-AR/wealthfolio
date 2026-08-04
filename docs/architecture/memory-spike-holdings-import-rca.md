# RCA: Multi-GB Memory Spike After Holdings CSV Import

**Status:** Root cause **CONFIRMED** by controlled reproduction: identical 51-ticker CSVs
differing only in the snapshot year — `0224` reproduces the runaway/OOM, `2024` stays under
200 MB. Fixes proposed (§4), not yet implemented. Awaiting the reporter's data only to confirm
which mangling produced their bad date.
**Last updated:** 2026-08-04 — re-verified against `feature/1347-performance-hotfix` @
`1775193`, incorporating corrections from an independent static review (see §7).
**Severity:** Critical — app becomes unusable, macOS kills the process
**Affected area:** Holdings import (CSV wizard, manual entry — desktop **and** web/server
handlers) → post-import refresh (market sync + valuation recalculation)

---

## 1. Context

### 1.1 Reports

| Report | Symptom | Scale |
|---|---|---|
| Direct user report (email, 2026-07) | Imported **51 tickers** via holdings CSV into a brokerage (HOLDINGS-mode) account. Import succeeded; during the post-import "refresh" the app climbed to **44.9 GB** RAM, macOS raised "system has run out of application memory", force-quit required. Heavy resource usage recurs on **every** subsequent account add / import until restart. | 44.9 GB |
| [#1347](https://github.com/wealthfolio/wealthfolio/issues/1347) | Memory impact and timeouts/slowdowns when viewing 5+ years of history. | GBs |
| [PR #1218](https://github.com/wealthfolio/wealthfolio/pull/1218) (author's motivation) | ~3k activities over ~5y: slow rendering, **1.5–2 GB** peak RAM on full history update. | 1.5–2 GB |

All three share one pipeline; the 44.9 GB case additionally requires the date-range defect
described below. Related design review:
[`valuation-performance-review-and-target-design.md`](./valuation-performance-review-and-target-design.md).

### 1.2 Why 51 tickers cannot explain 44.9 GB on its own

The post-import refresh materialises roughly **1 KB per asset per day** of history (per-day
snapshot clones + forward-filled quote structs + FX maps — see §2.3). For a sane single-date
holdings snapshot the range is days-to-weeks: 51 tickers compute to **tens of MB**. Even a
snapshot backdated 15 years stays under ~300 MB per pipeline run. The reported magnitude
therefore requires the *date range itself* to be pathological. 44.9 GB is not a working-set
size; it is where macOS killed a process whose allocation was growing without bound.

---

## 2. Root Cause Analysis

Three defects compose. The first supplies the unbounded input; the second and third turn it
into unbounded memory.

### 2.1 Primary: snapshot dates are format-validated but never range-validated

- `check_holdings_import` (`apps/tauri/src/commands/portfolio.rs`, ~line 1220 on the hotfix
  branch) validates CSV dates with `NaiveDate::parse_from_str(&snapshot.date, "%Y-%m-%d")` —
  **format only**. Years `0224`, `1924`, `2924`, `9999` all parse successfully. There is no
  "not in the future" and no lower-bound check.
- The same gap exists in `save_manual_holdings` (~1084) and `import_holdings_csv` (~1472), in
  the **web/server handlers** (`apps/server/src/api/holdings/handlers.rs` ~682 — same
  format-only pattern), and in `SnapshotService::save_manual_snapshot`
  (`crates/core/src/portfolio/snapshot/snapshot_service.rs` ~1534): no path rejects an absurd
  `snapshot_date`.
- There is also **no database-level constraint**: `holdings_snapshots.snapshot_date` is
  `DATE NOT NULL` with no `CHECK`, and **device-sync restore copies `holdings_snapshots` rows
  directly between SQLite databases**
  (`crates/storage-sqlite/src/sync/app_sync/repository.rs`, snapshot-restore path), bypassing
  the service layer entirely. Write-side validation (F1) therefore cannot be the only defence —
  a poisoned row can arrive via sync or pre-exist; the read-side clamp (F2) is the essential
  containment layer.
- Holdings CSV semantics make a single typo consequential: *"Rows with the same date form one
  snapshot; multiple dates create multiple snapshots"* (`portfolio.rs`, `import_holdings_csv`
  doc comment). **One mangled year in one of 51 hand-edited rows silently creates a second
  snapshot at that bogus date.** The import reports success — matching the reporter's "import
  was good".

### 2.2 The valuation window start is derived from stored dates with no floor

`get_daily_holdings_snapshots` (`crates/core/src/portfolio/snapshot/snapshot_service.rs`,
~1395-1425 on the hotfix branch) derives the reconstruction window as:

```
start = earliest non-synthetic snapshot_date              -- NO floor
end   = (latest snapshot ≤ today+365 days).max(today)     -- effectively capped near today
```

**Past dates are the OOM vector; far-future dates are not.** (Correction 2026-08-04 — an
earlier revision of this document claimed `end = MAX(snapshot_date).max(today)` with no
ceiling. That was wrong: the end-date lookup only considers snapshots up to `today + 365
days`, so a year-2924/9999 snapshot is not found by it. A lone far-future snapshot yields
empty/missing history, not a huge walk; a future date within 365 days extends the window by at
most a year — benign. All measured reproductions used past dates and are unaffected by this
correction.)

A snapshot at year 0224 sets `start = 0224-07-20` (≈ 658,000 days to today) with only the
`start > end → return empty` guard, and poisons the window for **every subsequent
recalculation** of that account. The floor is the missing half; the ceiling effectively exists.

### 2.3 The pipeline materialises O(days × positions-in-poisoned-snapshot) in memory

For the poisoned window, `calculate_valuation_history`
(`crates/core/src/portfolio/valuation/valuation_service.rs:1661`) allocates, simultaneously:

| Allocation | Source | Scaling |
|---|---|---|
| One deep-cloned `AccountStateSnapshot` **per calendar day** | `get_daily_holdings_snapshots`, `snapshot_service.rs:1313` | days × **positions carried on that day** (~400 B + map overhead per position) |
| One forward-filled `Quote` struct (~6 heap `String`s) per asset per day — **but only from each ticker's first real quote onward** (`last_known_quotes` starts empty) | `fill_missing_quotes`, `crates/core/src/quotes/service.rs:2394-2435` | assets × real quote history (~300 MB for 51 tickers), **not** the poisoned window |
| Per-day FX rate maps | `fetch_fx_rates_for_range`, `valuation_service.rs` (~417 on the hotfix branch) | **walks the FULL poisoned range** (one BFS per day × currency pair + one map entry per day) for multi-currency accounts; empty for single-currency accounts (the USD-only repro exercised no FX cost) |
| Per-day cloned quote maps, output valuation rows | `valuation_service.rs` (~1921) | bounded by quote-covered days |

The dominant term is the per-day snapshot clone, and it scales with **how many positions the
bad-dated snapshot carries** across the poisoned prefix (the pre-first-quote days are skipped
for valuation output by quote gating, but the snapshot clones happen before gating):

| Scenario (658k-day window) | Per-day clone | Per pipeline | × 2 pipelines |
|---|---|---|---|
| **One** mangled row (1 position carried) | ~1 KB | ~0.7 GB | **~1.5–2 GB** — *measured: 2 GB spike, settles at 600 MB* |
| **Whole date column** mangled (51 positions carried) | ~24 KB | ~16 GB | **~32 GB+** — *measured: runaway until the macOS OOM dialog* |

The reporter's 44.9 GB therefore implies the **entire date column** was mangled (an
export/locale/spreadsheet reformat affecting every row identically), not a single-row typo.
`fetch_fx_rates_for_range` additionally performs a graph BFS per (day × currency-pair), so the
refresh also spins CPU — matching "application was trying to refresh data".

### 2.4 Amplifier: the import triggers TWO independent full pipelines (overlap likely, not guaranteed)

A holdings import schedules the refresh twice, through two mechanisms that share no guard:

1. `import_holdings_csv` / `save_manual_holdings` explicitly emit
   `PORTFOLIO_TRIGGER_RECALCULATE` (`portfolio.rs:1160-1170`) → `listeners.rs`
   `handle_portfolio_request` → market sync + `SnapshotRecalcMode::Full` +
   `ValuationRecalcMode::Full`. **`listeners.rs` spawns unconditionally — it has no
   `is_processing` guard** (`apps/tauri/src/listeners.rs:65`).
2. `SnapshotService::save_manual_snapshot` emits `DomainEvent::HoldingsChanged`
   (`snapshot_service.rs:1399-1405`) → domain-event queue → planner maps it with
   `since_date = None` (`apps/tauri/src/domain_events/planner.rs:118-136, 194-198`) →
   `run_portfolio_job` maps `None → Full` (`queue_worker.rs:317-324`). The queue worker's
   `is_processing` guard does not see pipeline (1).

Whether the two runs overlap depends on timing (the queue debounce is 1000 ms —
`DEBOUNCE_MS`, `queue_worker.rs:39` — plus market-sync duration), so in the general case this
is a *possible* ~2× amplifier, not a guaranteed one. **In the OOM scenario specifically,
overlap is effectively certain:** the first pipeline's poisoned valuation walk runs for
minutes-to-never, so the second trigger always lands while it is still allocating. One
poisoned run is ~16 GB by itself; with overlap ~32 GB+ — the reported 44.9 GB is where macOS
killed the process (macOS "App Memory" also counts compressed pages). The double-trigger also
makes *sane* imports pay double (two market syncs + two Full recalcs), and the persistence of
the poisoned snapshot explains the recurrence: every later import, and app startup's backfill
check (`apps/tauri/src/lib.rs:188`), re-enters the poisoned account.

### 2.5 Verification status

**Confirmed in code:** the validation gap; the unclamped window; the per-day materialisation
sizes; the double-trigger with no shared guard; the `since_date=None → Full` planner mapping.

**Reproduced and controlled on a real Mac (this investigation):**

| File | Only difference | Window | Measured |
|---|---|---|---|
| `holdings-control-2024.csv` | all rows dated `2024-07-20` | ~740 days | **< 200 MB**, refresh completes normally |
| `holdings-repro-oom.csv` | **one** row dated `0224-07-20` | ~658k days × 1 position | **2 GB spike**, settles at 600 MB |
| `holdings-repro-oom-v2.csv` | **all** rows dated `0224-07-20` | ~658k days × 51 positions | **runaway → macOS OOM** (reporter's scenario) |

The control and the OOM file are byte-identical except the date field (same tickers,
quantities, prices, cash), so the snapshot date is isolated as the sole cause. Memory scales
with (window length × positions carried), exactly as modelled in §2.3.

**Remaining open item (does not affect the root cause):** which mangling produced the
reporter's bad date. The controlled experiment proves the mechanism and the magnitude; the
whole-column variant matches their 44.9 GB. Diagnostic to send the reporter:

```sql
SELECT account_id, snapshot_date FROM holdings_snapshots ORDER BY snapshot_date ASC  LIMIT 3;
SELECT account_id, snapshot_date FROM holdings_snapshots ORDER BY snapshot_date DESC LIMIT 3;
```

Any date outside the user's real investing horizon confirms the diagnosis. If their dates are
clean, the same unclamped-window mechanism still holds but the bad date entered through another
source (broker sync, device sync) — the clamp in fix F2 covers all of them.

**Ruled out during analysis:** per-holding concurrency (the frontend sends one command with all
51 holdings); a health-check auto-fix loop (`rebuild_account_history` is user-initiated); the
net-worth quote grid (alternative assets only).

---

## 3. Reproduction

A family of CSVs in the holdings-wizard format (51 real tickers + `$CASH`, identical
quantities/prices from a fixed seed), differing **only in the date field**. This makes the
experiment a controlled comparison: the control and the OOM file are byte-identical except the
year, so the observed memory difference is attributable to the date alone.

| File | Date field | Window | Result |
|---|---|---|---|
| `holdings-control-2024.csv` | all rows `2024-07-20` | ~740 days | **Measured: < 200 MB**, completes in seconds |
| `holdings-repro-mild.csv` / `holdings-dose-1924.csv` | `1924-07-20` (one row / all rows) | ~37k days | intermediate dose point, not run (unnecessary once the control confirmed) |
| `holdings-repro-oom.csv` | **one** row at `0224-07-20` | ~658k days × 1 position | **Measured: 2 GB spike → settles at 600 MB** |
| `holdings-repro-oom-v2.csv` | **all 52 rows** at `0224-07-20` | ~658k days × 51 positions | **Measured: runaway → macOS OOM dialog** — the reporter's scenario |

Steps: back up the DB → new HOLDINGS-mode account per file → import via the holdings CSV wizard
(observe: **no validation error on the mangled dates**) → watch Activity Monitor as the
post-import refresh starts.

**Cleanup between runs and after testing (important):** the poisoned rows persist and re-poison
every later refresh of their account — including on the tester's own machine. Delete the test
accounts, or run:
`DELETE FROM holdings_snapshots WHERE snapshot_date < '1990-01-01';` then recalculate.

---

## 4. Proposed Fixes

In priority order (revised 2026-08-04: **F2 is now P0** — write-side validation cannot reach
rows arriving via device-sync's direct table copy or already present in existing databases).
F1–F3 are small, independent patches; F4 is the structural fix tracked in the design document.

### F2 — Clamp the valuation window defensively (P0 — contains any bad date, from any source)

In `get_daily_holdings_snapshots` (and the valuation range derivation), clamp the start to a
floor and — because a 1900 floor alone still admits a ~46,000-day walk — enforce a **maximum
reconstruction span / resource budget** (e.g., reject or truncate windows beyond ~N×10k days
with a loud error) *before* allocating the date and snapshot vectors. **Log loudly** when a
stored date falls outside the clamp instead of silently walking the range. This is the only
layer that protects synced, restored, and pre-existing data.

### F1 — Range-validate snapshot dates at every write path (closes the door)

Reject `snapshot_date > today` and `snapshot_date < 1900-01-01` (constant, with a clear user
message). **Centralize the check in core** (`SnapshotService::save_manual_snapshot` /
`ManualSnapshotService`) so desktop and web share it, then surface per-row errors in:
- `check_holdings_import` (Tauri) and the equivalent web handler
  (`apps/server/src/api/holdings/handlers.rs`) — wizard-level validation errors,
- `import_holdings_csv` / `save_manual_holdings` — defence in depth at the command layer.

Note: F1 does **not** cover device-sync restore, which copies `holdings_snapshots` rows
directly at the storage layer, nor rows already in user databases — that is F2's job. A
`CHECK` constraint on `snapshot_date` is worth considering alongside, with a data migration
for existing offenders.

### F3 — Deduplicate the post-import trigger (halves every import's cost)

Remove the explicit `emit_portfolio_trigger_recalculate` from `import_holdings_csv` and
`save_manual_holdings` and rely on the guarded domain-event queue job. Alternatively (or
additionally), give `listeners.rs` the same `is_processing` guard so direct Tauri triggers and
queue jobs cannot run the pipeline concurrently.

Follow-on in the same area: add `earliest_snapshot_date` to `HoldingsChanged` (or consolidate
it with `ManualSnapshotSaved`) so imports schedule `SinceDate` instead of `None → Full`
(`planner.rs`). **Do not blanket-assign a since-date to `AccountsChanged`** — account deletion
or currency changes can legitimately require a full rebuild.

### F4 — Structural: stop materialising O(days × assets) (design doc WS-B)

Even with sane dates the refresh allocates ~1 KB per asset-day. The design document's WS-B
(interval-based valuation, `(asset, date) → close` numeric map shared across accounts, bounded
account fan-out instead of unbounded `join_all` in `listeners.rs:342`) reduces this to
O(assets) working memory and directly addresses #1347 and the PR #1218 author's 1.5–2 GB
report. See
[`valuation-performance-review-and-target-design.md`](./valuation-performance-review-and-target-design.md),
§4.3.

### Workaround for affected users (no release required)

Delete the bad-dated snapshot row (SQL above, or the snapshot UI once visible) and trigger a
recalculation. Memory returns to normal immediately; the poisoned row is the persistent state
that makes the problem recur.

---

## 5. Next Steps

1. **Implement F1–F3** (small, independent patches — none started yet):
   F1 turns this whole bug class into a friendly per-row validation error in the import wizard;
   F2 protects every existing database that may already contain a bad date from any source;
   F3 halves the cost of every import, sane or not.
2. **Reply to the reporter** with: (a) the workaround —
   `DELETE FROM holdings_snapshots WHERE snapshot_date < '1990-01-01';` (adjust the bound if
   their bad date is in the future: `... > date('now')`) then recalculate, memory returns to
   normal immediately; (b) the diagnostic SQL from §2.5, or simply ask for their CSV's date
   column — if their bad snapshot carries all 51 positions, the last open item closes;
   (c) confirmation that a validation fix is planned.
3. **#1347 status (updated 2026-08-04):** partially addressed by
   `feature/1347-performance-hotfix` @ `1775193`, which removed transfer-flow bottlenecks
   (sparse market facts via `transfer_market_facts_for_activities` instead of range-filled
   quote lookups) — so "all reports share one unchanged pipeline" is no longer precise. The
   **core import valuation path remains dense** (`get_daily_holdings_snapshots` →
   `get_quotes_in_range_filled` → per-day materialisation, `valuation_service.rs:1791-1921` on
   that branch), so F4/WS-B still applies to it. PR #1218 and its follow-up (including the
   storage-reclaim migration) are also merged on that branch.
4. **Regression tests** alongside F1/F2: import wizard rejects out-of-range dates; valuation
   window stays clamped even with a hostile `snapshot_date` planted in the repository.

---

## 6. Timeline / Links

- [PR #1218](https://github.com/wealthfolio/wealthfolio/pull/1218) — snapshot bloat fix; same
  pipeline, sane-date magnitude (1.5–2 GB).
- [#1347](https://github.com/wealthfolio/wealthfolio/issues/1347) — 5+ year history memory and
  timeout reports.
- Direct user report (2026-07) — 44.9 GB spike on 51-ticker holdings CSV import; this RCA.
- Controlled reproduction (2026-07/08, this RCA §2.5/§3) — control < 200 MB; single bad row
  2 GB; whole bad column → OOM. Root cause confirmed.
- `feature/1347-performance-hotfix` @ `1775193` — transfer-flow bottleneck fixes for #1347;
  PR #1218 + follow-up merged. Verified reference for this document's line numbers.

---

## 7. Review & Verification Log

**2026-08-04 — independent static review, verified claim-by-claim against `1775193`.**
An external review assessed this RCA against `feature/1347-performance-hotfix`. Every claim was
re-checked in code; results:

- **Accepted and applied:** the future-date claim was overstated (the end-date lookup caps at
  `today + 365 days`; far-future dates yield empty history, past dates remain the vector —
  §2.2 corrected); pipeline overlap is timing-dependent in general (§2.4 softened, with the
  note that overlap is effectively certain in the OOM case because the first run never
  completes); FX prefetch walks the full poisoned range for multi-currency accounts (§2.3
  corrected — it was previously listed as quote-bounded); the same validation gap exists in the
  **web/server handlers**, previously uncited (§2.1); device-sync restore bypasses the service
  layer and there is no DB constraint, so F1 alone is insufficient and **F2 is promoted to P0**
  with a max-span budget beyond the 1900 floor (§4); `AccountsChanged` should not be
  blanket-assigned a since-date (§4/F3); the #1347 relationship is updated for the hotfix
  branch (§5.3).
- **Reviewer errata (minor):** the queue debounce is **1000 ms** (`DEBOUNCE_MS`,
  `queue_worker.rs:39`) — the 500 ms figure comes from a stale doc comment in the same file;
  and the FX full-range walk costs nothing for single-currency accounts (the USD-only repro),
  which is why the measured repro numbers needed no revision.
- **Repository hygiene:** this RCA and the companion design document live on the
  `claude/valuation-calculations-performance-ii3pvr` branch, not on the hotfix branch the
  reviewer examined — hence "missing from repository" from their vantage point. The
  reproduction CSVs were session artifacts; they are now committed under
  `docs/test-data/memory-repro/`.
- **Unchanged:** all measured results (§2.5/§3) and the root-cause conclusion. The reviewer's
  bottom line — F1/F2/F3 remain necessary, F2 first — matches this document's revised order.
- [`valuation-performance-review-and-target-design.md`](./valuation-performance-review-and-target-design.md)
  — full architecture review; §2.2 (compute hot spots) and §4.3 (WS-B) are the structural
  context for F4.
