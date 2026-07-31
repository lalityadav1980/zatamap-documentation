# NIFTY Runner Coverage Regression Gate

This gate changes the investigation workflow from day-by-day tuning to benchmark-driven regression.
The trading engine can still be improved day by day, but every code change must now be measured against documented Runner and Major Runner legs before it is trusted.

## Objective

The benchmark documents describe the NIFTY index opportunity legs. The gate compares those legs with the actual option trades produced by a backtest export and classifies each tradable leg as:

| Status | Meaning |
|---|---|
| `CAPTURED` | Same-side option trade covered the leg within configured entry, exit, and window-coverage limits. |
| `EARLY_EXIT` | Same-side trade entered the leg but exited too far before the benchmark leg ended. |
| `LATE_ENTRY` | Same-side trade entered after the allowed delay window. |
| `LOW_WINDOW_CAPTURE` | Same-side trade overlapped the leg, but covered too little of the benchmark window. |
| `WRONG_SIDE_OVERLAP` | Opposite-side trade overlapped the benchmark leg while no same-side trade covered it. |
| `MISSED` | Replay completed the leg and no matching same-side trade was found. |
| `MISSED_SO_FAR` | Replay is still inside the leg and no matching same-side trade exists yet. |
| `OPEN_PARTIAL` | Same-side trade is open while replay is still inside the benchmark leg. |
| `PENDING_REPLAY` | Replay has not reached the benchmark leg yet. |

## Usage

Export the trade tape from the backtest DB:

```sql
SELECT o.order_timestamp,
       o.parent_order_id,
       o.tradingsymbol,
       o.transaction_type,
       o.tag,
       o.matched_pnl,
       t.max_pct,
       t.min_pct,
       t.exit_reason
FROM trade.orders_book o
LEFT JOIN trade.trail_summary_day t
  ON t.parent_order_id = o.parent_order_id
ORDER BY o.id ASC;
```

For a completed replay, run:

```bash
python3 tools/benchmark_runner_gate.py \
  --benchmark-md docs/analysis/2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md \
  --benchmark-md docs/analysis/2026-07-04-nifty-raw-benchmark-2026-06-16-to-2026-06-30.md \
  --benchmark-md docs/analysis/2026-07-04-nifty-raw-benchmark-2026-07-01-to-2026-07-03.md \
  --orders-csv outputs/backtest_trade_tape.csv \
  --output-csv outputs/runner_coverage_gate.csv \
  --output-md outputs/runner_coverage_gate.md
```

For a focused export that contains only selected days, avoid false misses from other benchmark dates:

```bash
python3 tools/benchmark_runner_gate.py \
  --benchmark-md docs/analysis/2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md \
  --orders-csv outputs/four_day_trade_tape.csv \
  --only-order-dates \
  --output-csv outputs/four_day_runner_coverage_gate.csv \
  --output-md outputs/four_day_runner_coverage_gate.md
```

You can also pin exact dates:

```bash
python3 tools/benchmark_runner_gate.py \
  --benchmark-md docs/analysis/2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md \
  --orders-csv outputs/backtest_trade_tape.csv \
  --date 2026-04-29,2026-04-30 \
  --output-csv outputs/apr29_apr30_runner_coverage_gate.csv
```

For a still-running replay, include per-day progress so future legs are marked `PENDING_REPLAY` instead of `MISSED`:

```csv
date,max_event_ist
2026-04-30,2026-04-30 11:27:00+05:30
```

```bash
python3 tools/benchmark_runner_gate.py \
  --benchmark-md docs/analysis/2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md \
  --orders-csv outputs/backtest_trade_tape.csv \
  --events-csv outputs/replay_progress.csv \
  --output-csv outputs/runner_coverage_gate.csv \
  --output-md outputs/runner_coverage_gate.md
```

## Regression Policy

Before changing entry or exit logic:

1. Run the gate on the current baseline export.
2. Identify failures by class, not by date alone.
3. Read comments around the relevant trading-rule code before editing.
4. If a comment names a protected day, rerun that day after the change.
5. Rerun the full benchmark gate and compare failure counts before promoting code.

Default failing statuses are:

```text
MISSED, WRONG_SIDE_OVERLAP, LATE_ENTRY, EARLY_EXIT, LOW_WINDOW_CAPTURE
```

`OPEN_PARTIAL`, `MISSED_SO_FAR`, and `PENDING_REPLAY` are not failures because they can occur during an active replay.

## Why This Matters

The engine has historically optimized option-specific gates, P&L locks, and retry paths. That can make one day better while quietly breaking another day. This gate makes the documented NIFTY runner legs the regression target, so a fix must improve a failure class without degrading protected runner coverage elsewhere.
