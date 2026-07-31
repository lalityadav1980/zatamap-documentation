# 2026-06-29 PE Runner Fix Validation

Local validation run only. No production host or live instance was changed.

## Root Cause

The 2026-06-29 PE move was not missed because market data was absent. DVR saw the PE recovery, but by the time the token looked good to the single-token DVR detector it was classified as `reject_exhausted_recovery` / `recovery_already_extended`.

The structural gap was in the handoff from DVR/OPTION_LED into STABLE_RETRY:

- `SIDEWAYS -> PE` selected by OPTION_LED was not treated as side-aligned by candidate reservation.
- The candidate reservation path required a materialized reserved candidate, but the broad PE basket runner could remain blocked as an extended above-VWAP chase.
- The STABLE_RETRY CDE path could receive the wrong SIDEWAYS fallback side instead of the actual desired PE side.
- STABLE_RETRY order debounce could start before a real order attempt, making a rejected path suppress later valid attempts.
- The trailing ladder could close fresh STABLE_RETRY/MOMENTUM runners on low-rung noise before structural handover confirmed.

## Fix Summary

- Added strict same-side basket-runner handoff for STABLE_RETRY.
- Allowed OPTION_LED `SIDEWAYS -> PE/CE` to count as side-aligned for candidate reservation.
- Materialized basket-runner candidates instead of silently dying as `entry_without_materialized_reserved_candidate`.
- Passed the actual desired side into CDE for STABLE_RETRY.
- Started STABLE_RETRY debounce only after an actual order attempt.
- Suppressed low-rung ladder exits for fresh sponsored STABLE_RETRY/MOMENTUM runners unless L2, held-token rollover, and spot transfer confirm structural handover.

## Local Replay Evidence

Run ID: `phase3-0629-fullstate-fixed-v14-1530`

Command class: `tools/run_timescale_multiday_backtest.py`, local DB `127.0.0.1:5432/zatamap_market`.

Summary:

| Metric | Value |
|---|---:|
| Status | `ok` |
| Return code | `0` |
| Market window | `2026-06-29 09:15:00` to `15:30:00` IST |
| Market rows | `3,734,871` |
| Orders | `8` |
| Buys | `4` |
| Sells | `4` |
| Open tracker rows | `0` |
| Realized PnL | `+96,369.00` |
| Matched PnL | `+96,369.00` |

Order sequence:

| Time IST | Symbol | Side | Qty | Avg | Tag | Matched PnL |
|---|---|---|---:|---:|---|---:|
| 09:32:19 | NIFTY26JUN24100PE | BUY | 1755 | 122.35 | TR_STABLE_RETRY | 0.00 |
| 09:32:19 | NIFTY26JUN24100PE | BUY | 520 | 122.35 | TR_STABLE_RETRY | 0.00 |
| 09:38:28 | NIFTY26JUN24100PE | SELL | 1755 | 117.90 | EXIT_LADDER_LOCK | -7,809.75 |
| 09:38:28 | NIFTY26JUN24100PE | SELL | 520 | 117.90 | EXIT_LADDER_LOCK | -2,314.00 |
| 11:53:48 | NIFTY26JUN24100PE | BUY | 1755 | 133.00 | TR_STABLE_RETRY | 0.00 |
| 11:53:48 | NIFTY26JUN24100PE | BUY | 260 | 133.00 | TR_STABLE_RETRY | 0.00 |
| 12:37:37 | NIFTY26JUN24100PE | SELL | 1755 | 185.85 | EXIT_TARGET_VWAP | +92,751.75 |
| 12:37:37 | NIFTY26JUN24100PE | SELL | 260 | 185.85 | EXIT_TARGET_VWAP | +13,741.00 |

Key event sequence:

| Time IST | Event |
|---|---|
| 11:53:48 | `STABLE_RETRY_BASKET_RUNNER_HANDOFF` cleared whipsaw participation gate |
| 11:53:48 | `CANDIDATE_RESERVATION: would_materialize reason=basket_runner_handoff` |
| 11:53:48 | `CANDIDATE_RESERVATION: entry_would_allow reason=entry_matches_materialized_candidate` |
| 11:53:48 | `BUY.fill COMPLETE` for `NIFTY26JUN24100PE` |
| 12:37:37 | `CLOSE.done tag=EXIT_TARGET_VWAP` |

## Raw Opportunity Evidence

NIFTY larger PE leg: `11:04` to `12:41`, approximately `24081.55 -> 23928.15`, `-153.40` points from local first/last raw ticks in that window.

PE movers during `11:04-12:41`:

| Symbol | First LTP | Max LTP | Max Move |
|---|---:|---:|---:|
| NIFTY26JUN23850PE | 18.05 | 57.90 | +220.78% |
| NIFTY26JUN23900PE | 25.00 | 76.80 | +207.20% |
| NIFTY26JUN24050PE | 65.00 | 160.40 | +146.77% |
| NIFTY26JUN24100PE | 87.55 | 196.95 | +124.96% |

## Local Report

Self-contained local HTML report:

`.trade-api-runs/phase3-0629-fullstate-fixed-v14-1530/2026-06-29/report.html`

The `.trade-api-runs/` directory is intentionally not committed because it contains generated replay artifacts and logs.

## Caveat

The local laptop DB is missing auxiliary replay tables such as `trade.orders_book_live`, `trade.positions`, and margin snapshot tables. The replay logs contain non-fatal upsert tracebacks for those tables after fills, but the canonical `trade.orders_book` and `trade.orders_tracker` state completed cleanly with `rc=0` and tracker count `0`.
