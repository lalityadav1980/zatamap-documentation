# Phase 3 Index Runner Ladder Hold - 2026-07-05

This note records the first behavior-changing fix on the runner coverage consolidation branch.

## Failure Class

`EARLY_EXIT`

The remote runner coverage workshop showed that several trades entered the correct side of a documented NIFTY `Runner` or `Major Runner`, but exited through `EXIT_LADDER_LOCK` while the same-side NIFTY runner was still alive.

Representative baseline failures:

| Date | Benchmark leg | Side | Baseline entry | Baseline exit | Baseline exit reason |
|---|---|---|---|---|---|
| 2026-04-29 | 09:23-10:28, +231.5 pts | CE | 09:27:37 | 10:02:48 | `EXIT_LADDER_LOCK` |
| 2026-05-13 | 09:40-10:34, +207.0 pts | CE | 09:52:36 | 09:56:02 | `EXIT_LADDER_LOCK` |
| 2026-05-26 | 11:39-14:54, -181.7 pts | PE | 11:49:15 | 12:04:02 | `EXIT_LADDER_LOCK` |

## Structural Fix

Add `index_runner_state.py` and call `evaluate_index_runner_state()` from the `EXIT_LADDER_LOCK` path in `order_service_api.py`.

The detector uses only live/replay spot tick history available at the current timestamp. It does not read benchmark labels or future data.

The hold is symmetric:

| Held option | Required index structure |
|---|---|
| CE | NIFTY has an active bullish runner from a recent anchor low. |
| PE | NIFTY has an active bearish runner from a recent anchor high. |

The hold only defers ladder-lock style exits. It does not disable hard stop-loss, EOD close, or proven structural release conditions.

## Release Conditions

The runner hold does not apply when:

| Condition | Why |
|---|---|
| Same-side index runner is inactive or invalidated | No runner lifecycle remains to protect. |
| Option giveback exceeds bounded cap | Prevents "hold forever" behavior. |
| Existing runner floor release has fired | Preserves historical chop protections. |
| Entry is not a runner/recovery/momentum family | Avoids widening exits for unrelated scalp/range trades. |
| The trade is classified as range scalp | Scalp/sideways trading is not the objective of this phase. |

## Local Validation

Focused local replay:

| Field | Value |
|---|---|
| Run id | `replay-lab-fast-20260705` |
| Replay day | 2026-04-29 |
| User id | `bt_replay_lab_fast_20260705` |
| Entry | 09:27:37 `NIFTY2650524000CE` BUY @ 258.30 |
| Exit | 10:30:23 SELL @ 400.60 |
| Exit reason | `EXIT_TARGET_VWAP` |
| Matched PnL | +147,992 |
| Max option PnL | +61.17% |
| Min option PnL | -3.12% |

This fixes the observed 04-29 early-exit class locally: the trade stayed open through the previous 10:02 ladder exit point and exited just after the 09:23-10:28 benchmark Major Runner completed.

Coverage-gate result while the replay was still in progress:

| Benchmark leg | Baseline status | Candidate status | Coverage |
|---|---|---|---|
| 2026-04-29 leg 4, CE Major Runner 09:23-10:28 | `EARLY_EXIT` | `CAPTURED` | 0.93 |
| 2026-04-29 leg 7, PE Major Runner 13:08-14:30 | not yet scored | `PENDING_REPLAY` | - |

## Next Regression Requirement

Before promotion, run the protected early-exit regression group:

| Date | Why |
|---|---|
| 2026-04-29 | Primary CE major-runner early-exit fix. |
| 2026-05-13 | CE DVR recovery early-exit case. |
| 2026-05-26 | PE long major-runner early-exit case. |
| 2026-04-20 | Whipsaw/reversal protection. |
| 2026-04-21 | Same-side profit reentry guard protection. |
| 2026-05-04 | High-opportunity whipsaw protection. |

Promotion should be based on benchmark coverage status changes, not only PnL.
