# Phase 0 Local Replay Lab Fix - 2026-07-05

## Problem

The focused local replay for known trade days returned zero orders. That result was not a strategy result; the replay lab was not starting the strategy correctly.

## Evidence

The broken run logs showed:

```text
Profile debug: no rows in trade.profile for userid=bt_index_runner_phase1
Profile data not found for bt_index_runner_phase1
APP_DB_BACKTEST_DAY_DONE
```

The runner had cloned the isolated profile into `zatamap_market.trade.profile`, but the replayed app process still launched with:

```text
PostgreSQL pool created: host=localhost db=zatamap_trade user=zatamap
```

So the app was reading profile/config from `zatamap_trade`, while the backtest runner had prepared data in `zatamap_market`.

## Fix

`tools/run_timescale_multiday_backtest.py` now defaults `--pg-database` to `zatamap_market`, matching the default `--trade-dsn`.

This keeps profile lookup, simulated orders, tracker rows, fusion events, and trail tables in the same local DB that the UI/debug queries use.

## Validation

After the fix, the local replay for 2026-04-29:

1. Fetched profile successfully.
2. Initialized the backtest runtime in DB mode.
3. Subscribed replay ticks.
4. Generated fusion events.
5. Opened a CE order at 09:27:38 IST.

The validation run also showed the local candidate's runner-exit prototype holding past the old remote baseline early-exit point, with logs like:

```text
TSL_15ONLY.INDEX_RUNNER_LADDER_HOLD ... EXIT_LADDER_LOCK deferred because same-side NIFTY runner is active
```

That behavior still needs a completed day replay and benchmark classification before promotion.

## Guardrail

A unit test now verifies that the replayed app defaults to `zatamap_market`, while still respecting an explicit `PGDATABASE` override.
