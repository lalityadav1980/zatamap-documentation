# 2026-04-29 R28 Backtest Benchmark

Source run:
`full-2026-04-16-to-2026-06-08-vwap-session-runner-r28-20260616_162655`

Purpose: preserve 29-Apr as the working control day before testing the new
toggle-based whipsaw participation gate on 20-Apr.

## Raw Market Benchmark

2026-04-29 was classified as `ROUND_TRIP_MIXED_BIAS / CHOPPY_ALTERNATING`.

Important benchmark legs:

| Leg | Time | Direction | Spot Move | Runner Class |
| --- | --- | --- | ---: | --- |
| 4 | 09:23-10:28 | Bull | +231.5 pt | Major Runner |
| 7 | 13:08-14:30 | Bear | -145.3 pt | Major Runner |
| 9 | 14:44-15:26 | Bear | -78.0 pt | Slow Tradable Leg |

## Captured Trades

| Trade | Side | Token | Entry | Exit | Qty | Entry | Exit | PnL | Return | Exit |
| --- | --- | --- | --- | --- | ---: | ---: | ---: | ---: | ---: | --- |
| T1 | CE | NIFTY2650524000CE | 09:27:38 | 10:30:21 | 2600 | 258.00 | 398.85 | +366,210.00 | +54.59% | EXIT_TARGET_VWAP |
| T2 | PE | NIFTY2650524350PE | 13:41:03 | 14:38:58 | 3185 | 210.85 | 256.20 | +144,439.75 | +21.51% | EXIT_TARGET_VWAP |

Gross day PnL from completed trades: `+510,649.75`.

## Trail Summary

| Trade | Min % | Max % | Final % | Left On Table | Exit |
| --- | ---: | ---: | ---: | ---: | --- |
| T1 CE | -3.00% | +61.36% | +54.59% | 6.77% | EXIT_TARGET_VWAP |
| T2 PE | -1.87% | +26.68% | +21.51% | 5.17% | EXIT_TARGET_VWAP |

## Control Interpretation

This is the behavior we should protect while testing 20-Apr:

- The first trade materialized early enough in the 09:23-10:28 bull major runner.
- VWAP-defer allowed the CE to run beyond the 25% target and exit after sponsorship weakened.
- The second trade captured the afternoon bear major runner with acceptable drawdown.
- Both trades exited through `EXIT_TARGET_VWAP`, not stop loss.

The new 20-Apr whipsaw participation work must not break this control pattern:

- Do not block clean early CE sponsorship like T1.
- Do not force premature target exit when VWAP sponsorship remains strong.
- Do not allow late/chase entries that are already extended without participation.

## Exported Files

- `docs/analysis/exports/2026-04-29-r28-orders-book.csv`
- `docs/analysis/exports/2026-04-29-r28-trade-legs.csv`
- `docs/analysis/exports/2026-04-29-r28-trail-summary.csv`
