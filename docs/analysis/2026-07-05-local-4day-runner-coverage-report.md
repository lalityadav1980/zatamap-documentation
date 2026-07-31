# NIFTY Runner Coverage Gate

Legs scored: 11
Failing legs: 8

## Input Summary

| Item | Value |
|---|---|
| Orders CSV | `outputs/runner-analysis-0702/local_4days_orders.csv` |
| Events CSV | not supplied; each order-date is treated as complete through 15:30 IST |
| Benchmark docs | docs/analysis/2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md<br>docs/analysis/2026-07-04-nifty-raw-benchmark-2026-06-16-to-2026-06-30.md<br>docs/analysis/2026-07-04-nifty-raw-benchmark-2026-07-01-to-2026-07-03.md |
| Fail statuses | `EARLY_EXIT, LATE_ENTRY, LOW_WINDOW_CAPTURE, MISSED, WRONG_SIDE_OVERLAP` |
| Min window coverage | `0.45` |
| Max entry delay | `15.0 min` or `0.35` of leg duration, whichever is larger |
| Max early exit | `10.0 min before benchmark leg end` |

## Status Summary

| Status | Count |
|---|---:|
| CAPTURED | 3 |
| EARLY_EXIT | 1 |
| MISSED | 7 |

## Failing Legs

| Date | Leg | Side | Time | Points | Status | Trade | Note |
|---|---:|---|---|---:|---|---|---|
| 2026-05-07 | 2 | CE | 09:22-09:43 | 95.1 | MISSED |  | MISSED |
| 2026-05-07 | 10 | CE | 11:45-12:17 | 94.0 | EARLY_EXIT | TK260507-115223-9444 | EARLY_EXIT: exit_before_leg_end_min=12.1, coverage=0.39 |
| 2026-05-07 | 13 | PE | 12:43-12:57 | -85.7 | MISSED |  | MISSED |
| 2026-05-12 | 3 | PE | 09:19-10:05 | -116.8 | MISSED |  | MISSED |
| 2026-05-12 | 5 | PE | 10:20-13:11 | -165.1 | MISSED |  | MISSED |
| 2026-05-12 | 13 | PE | 14:48-15:14 | -116.2 | MISSED |  | MISSED |
| 2026-05-26 | 1 | CE | 09:15-09:40 | 100.6 | MISSED |  | MISSED |
| 2026-05-26 | 6 | PE | 11:39-14:54 | -181.7 | MISSED |  | MISSED |

## All Legs

| Date | Leg | Class | Side | Time | Points | Status | Coverage | Entry | Exit |
|---|---:|---|---|---|---:|---|---:|---|---|
| 2026-05-07 | 2 | Runner | CE | 09:22-09:43 | 95.1 | MISSED |  |  |  |
| 2026-05-07 | 7 | Runner | PE | 10:45-11:27 | -81.8 | CAPTURED | 0.79 | 10:53:42 | 11:30:28 |
| 2026-05-07 | 10 | Runner | CE | 11:45-12:17 | 94.0 | EARLY_EXIT | 0.39 | 11:52:22 | 12:04:53 |
| 2026-05-07 | 12 | Impulse Runner | CE | 12:33-12:43 | 124.8 | CAPTURED | 0.49 | 12:37:58 | 12:42:55 |
| 2026-05-07 | 13 | Impulse Runner | PE | 12:43-12:57 | -85.7 | MISSED |  |  |  |
| 2026-05-07 | 19 | Runner | PE | 14:33-15:11 | -84.6 | CAPTURED | 0.64 | 14:16:15 | 15:03:06 |
| 2026-05-12 | 3 | Runner | PE | 09:19-10:05 | -116.8 | MISSED |  |  |  |
| 2026-05-12 | 5 | Major Runner | PE | 10:20-13:11 | -165.1 | MISSED |  |  |  |
| 2026-05-12 | 13 | Runner | PE | 14:48-15:14 | -116.2 | MISSED |  |  |  |
| 2026-05-26 | 1 | Runner | CE | 09:15-09:40 | 100.6 | MISSED |  |  |  |
| 2026-05-26 | 6 | Major Runner | PE | 11:39-14:54 | -181.7 | MISSED |  |  |  |
