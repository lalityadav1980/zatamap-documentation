# Remote Runner Coverage Workshop - 2026-07-05

This is the structural workshop baseline from the currently running 173 backtest. It uses the documented NIFTY benchmark legs as the contract and the remote backtest trades as the measured behavior.

The goal is to stop fixing isolated dates and instead fix repeated failure classes: `EARLY_EXIT`, `LATE_ENTRY`, `MISSED`, `WRONG_SIDE_OVERLAP`, and `LOW_WINDOW_CAPTURE`.

## Inputs

| Item | Value |
|---|---|
| Remote source | 173 box, read-only DB export from `trade.orders_book`, `trade.trail_summary_day`, and `trade.fusion_events` progress |
| Orders export | `outputs/runner-workshop/remote-current/orders_trail.csv` |
| Replay progress export | `outputs/runner-workshop/remote-current/replay_progress.csv` |
| Coverage CSV | `outputs/runner-workshop/remote-current/runner_coverage_progress.csv` |
| Coverage report | `outputs/runner-workshop/remote-current/runner_coverage_progress.md` |
| Benchmark docs | `docs/analysis/2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md`, `docs/analysis/2026-07-04-nifty-raw-benchmark-2026-06-16-to-2026-06-30.md`, `docs/analysis/2026-07-04-nifty-raw-benchmark-2026-07-01-to-2026-07-03.md` |

## Baseline Status

| Status | Count |
|---|---|
| CAPTURED | 10 |
| EARLY_EXIT | 7 |
| LATE_ENTRY | 9 |
| LOW_WINDOW_CAPTURE | 2 |
| MISSED | 33 |
| WRONG_SIDE_OVERLAP | 7 |
| OPEN_PARTIAL | 5 |
| MISSED_SO_FAR | 4 |
| PENDING_REPLAY | 32 |

Actionable failures only:

| Failure class | Count |
|---|---|
| MISSED | 33 |
| LATE_ENTRY | 9 |
| EARLY_EXIT | 7 |
| WRONG_SIDE_OVERLAP | 7 |
| LOW_WINDOW_CAPTURE | 2 |

Top failure dates:

| Date | Actionable failures | Breakdown |
|---|---|---|
| 2026-04-23 | 4 | MISSED=4 |
| 2026-05-11 | 4 | MISSED=2, LATE_ENTRY=1, WRONG_SIDE_OVERLAP=1 |
| 2026-05-21 | 4 | MISSED=2, WRONG_SIDE_OVERLAP=1, LATE_ENTRY=1 |
| 2026-04-27 | 3 | MISSED=2, LOW_WINDOW_CAPTURE=1 |
| 2026-04-28 | 3 | MISSED=3 |
| 2026-05-05 | 3 | MISSED=1, EARLY_EXIT=1, LATE_ENTRY=1 |
| 2026-05-08 | 3 | EARLY_EXIT=2, MISSED=1 |
| 2026-05-13 | 3 | EARLY_EXIT=1, MISSED=1, LATE_ENTRY=1 |
| 2026-05-15 | 3 | MISSED=2, EARLY_EXIT=1 |
| 2026-05-18 | 3 | WRONG_SIDE_OVERLAP=2, LATE_ENTRY=1 |
| 2026-05-20 | 3 | MISSED=3 |
| 2026-04-17 | 2 | MISSED=2 |
| 2026-04-24 | 2 | LATE_ENTRY=1, WRONG_SIDE_OVERLAP=1 |
| 2026-04-30 | 2 | MISSED=1, WRONG_SIDE_OVERLAP=1 |
| 2026-05-04 | 2 | MISSED=1, LOW_WINDOW_CAPTURE=1 |
| 2026-05-12 | 2 | MISSED=1, WRONG_SIDE_OVERLAP=1 |
| 2026-05-19 | 2 | MISSED=2 |
| 2026-05-26 | 2 | MISSED=1, EARLY_EXIT=1 |

## Major-Runner Failures

These are highest priority because they represent the largest documented legs.

| Date | Leg | Side | Time | Pts | Status | Entry | Exit | Entry tag | Exit reason | Note |
|---|---|---|---|---|---|---|---|---|---|---|
| 2026-04-16 | 4 | PE | 11:08-12:07 | -173.7 | LATE_ENTRY | 11:46:16 | 12:08:51 | TR_STABLE_RETRY_BASKET | EXIT_TARGET_VWAP | LATE_ENTRY: delay_min=38.3, limit_min=20.6, coverage=0.35 |
| 2026-04-20 | 3 | CE | 09:48-10:40 | 164.8 | LATE_ENTRY | 10:09:06 | 10:33:29 | TR_STABLE_RETRY_BASKET | EXIT_TARGET_VWAP | LATE_ENTRY: delay_min=21.1, limit_min=18.2, coverage=0.47, opposite=TK260420-093512-1213:PE:09:35:11-09:49:27:NIFTY2642124550PE |
| 2026-04-23 | 4 | PE | 10:05-11:44 | -128.0 | MISSED | - | - | - | - | MISSED |
| 2026-04-29 | 4 | CE | 09:23-10:28 | 231.5 | EARLY_EXIT | 09:27:37 | 10:02:48 | TR_STABLE_RETRY | EXIT_LADDER_LOCK | EARLY_EXIT: exit_before_leg_end_min=25.2, coverage=0.54 |
| 2026-05-05 | 6 | PE | 09:55-10:57 | -131.5 | MISSED | - | - | - | - | MISSED |
| 2026-05-11 | 11 | PE | 13:54-15:19 | -189.8 | WRONG_SIDE_OVERLAP | - | - | - | - | WRONG_SIDE_OVERLAP: opposite=TK260511-134558-16259:CE:13:45:57-OPEN:NIFTY2651223850CE |
| 2026-05-12 | 5 | PE | 10:20-13:11 | -165.1 | WRONG_SIDE_OVERLAP | - | - | - | - | WRONG_SIDE_OVERLAP: opposite=TK260512-115425-9566:CE:11:54:24-12:01:36:NIFTY2651223450CE |
| 2026-05-13 | 4 | CE | 09:40-10:34 | 207.0 | EARLY_EXIT | 09:52:36 | 09:56:02 | DVR_RECOVERY_CE | EXIT_LADDER_LOCK | EARLY_EXIT: exit_before_leg_end_min=38.0, coverage=0.06, opposite=TK260513-093017-918:PE:09:30:16-09:40:02:NIFTY2651923450PE |
| 2026-05-13 | 8 | CE | 10:59-11:45 | 126.2 | LATE_ENTRY | 11:42:22 | 11:57:52 | TR_STABLE_RETRY_BASKET | TR_STABLE_RETRY_BASKET | LATE_ENTRY: delay_min=43.4, limit_min=16.1, coverage=0.06, opposite=TK260513-110015-6316:PE:11:00:14-11:16:40:NIFTY2651923500PE |
| 2026-05-18 | 4 | CE | 10:47-11:39 | 166.7 | LATE_ENTRY | 11:38:31 | 11:47:25 | TR_STABLE_RETRY_R | EXIT_LADDER_LOCK | LATE_ENTRY: delay_min=51.5, limit_min=18.2, coverage=0.01 |
| 2026-05-20 | 7 | CE | 10:49-11:50 | 133.9 | MISSED | - | - | - | - | MISSED |
| 2026-05-21 | 7 | PE | 11:28-12:41 | -137.5 | LATE_ENTRY | 12:27:02 | 12:43:17 | TR_STABLE_RETRY_R | TR_STABLE_RETRY_R | LATE_ENTRY: delay_min=59.0, limit_min=25.5, coverage=0.19 |
| 2026-05-26 | 6 | PE | 11:39-14:54 | -181.7 | EARLY_EXIT | 11:49:15 | 12:04:02 | DVR_RECOVERY_PE_STABLE_RETRY_R | EXIT_LADDER_LOCK | EARLY_EXIT: exit_before_leg_end_min=170.0, coverage=0.08 |

## Failure Class Diagnosis

### 1. EARLY_EXIT: option P&L lock wins before the index runner invalidates

| Date | Leg | Class | Side | Time | Pts | Entry | Exit | Min early | Entry tag | Exit | Max % | Min % |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 2026-04-29 | 4 | Major Runner | CE | 09:23-10:28 | 231.5 | 09:27:37 | 10:02:48 | 25.2 | TR_STABLE_RETRY | EXIT_LADDER_LOCK | 34.57 | -3.12 |
| 2026-05-05 | 11 | Runner | CE | 11:58-12:22 | 87.0 | 12:09:29 | 12:11:52 | 10.1 | DVR_RECOVERY_CE | EXIT_LADDER_LOCK | 8.49 | -3.11 |
| 2026-05-08 | 3 | Runner | CE | 09:45-10:30 | 96.8 | 09:54:00 | 10:16:45 | 13.2 | DVR_RECOVERY_CE_NEAR_RECLAIM_RUNNER | EXIT_LADDER_LOCK | 9.36 | -4.95 |
| 2026-05-08 | 6 | Runner | PE | 11:15-12:46 | -84.5 | 11:22:43 | 11:32:15 | 73.8 | DVR_RECOVERY_PE | EXIT_LADDER_LOCK | 7.71 | -0.32 |
| 2026-05-13 | 4 | Major Runner | CE | 09:40-10:34 | 207.0 | 09:52:36 | 09:56:02 | 38.0 | DVR_RECOVERY_CE | EXIT_LADDER_LOCK | 7.76 | 0.06 |
| 2026-05-15 | 6 | Runner | PE | 10:51-11:31 | -142.1 | 11:02:53 | 11:10:58 | 20.0 | DVR_RECOVERY_PE | EXIT_LADDER_LOCK | 18.97 | -0.18 |
| 2026-05-26 | 6 | Major Runner | PE | 11:39-14:54 | -181.7 | 11:49:15 | 12:04:02 | 170.0 | DVR_RECOVERY_PE_STABLE_RETRY_R | EXIT_LADDER_LOCK | 36.27 | -3.78 |

Diagnosis: the exit layer protects option profit using ladder/target locks, but it does not have a shared NIFTY-runner lifecycle objective. A held option can be closed by `EXIT_LADDER_LOCK` while the benchmark NIFTY leg is still alive. Existing code has several token/VWAP sponsorship holds, but those are local to the held option and do not answer the higher-level question: is the same-side index runner still structurally alive?

Structural fix: add an index-runner hold layer that can veto ladder/target exits only when the held side matches an active NIFTY runner and the exit has not proven runner invalidation. Release on opposite index structure, opposite basket sponsorship, held-token VWAP failure, or bounded giveback. This is not a wider ladder; it is an evidence-based hold/release state.

### 2. WRONG_SIDE_OVERLAP: opposite-side entries survive inside a documented runner

| Date | Leg | Class | Needed side | Time | Pts | Opposite trade |
|---|---|---|---|---|---|---|
| 2026-04-24 | 3 | Runner | PE | 10:06-11:17 | -97.7 | TK260424-110657-6718:CE:11:06:56-OPEN:NIFTY26APR23900CE |
| 2026-04-30 | 3 | Runner | PE | 09:46-10:17 | -140.9 | TK260430-094310-1691:CE:09:43:09-09:57:51:NIFTY2650523800CE |
| 2026-05-11 | 11 | Major Runner | PE | 13:54-15:19 | -189.8 | TK260511-134558-16259:CE:13:45:57-OPEN:NIFTY2651223850CE |
| 2026-05-12 | 5 | Major Runner | PE | 10:20-13:11 | -165.1 | TK260512-115425-9566:CE:11:54:24-12:01:36:NIFTY2651223450CE |
| 2026-05-18 | 2 | Runner | CE | 09:57-10:37 | 120.2 | TK260518-092645-706:PE:09:26:44-09:59:52:NIFTY2651923450PE |
| 2026-05-18 | 5 | Runner | PE | 11:39-12:00 | -90.7 | TK260518-113832-8613:CE:11:38:31-11:47:25:NIFTY2651923400CE |
| 2026-05-21 | 6 | Impulse Runner | CE | 11:17-11:28 | 102.4 | TK260521-111715-7336:PE:11:17:14-11:20:48:NIFTY26MAY23750PE |

Diagnosis: the engine can recognize local option recoveries on the opposite side while the broader NIFTY runner is still active. That creates CE entries inside PE runners or PE entries inside CE runners. This is an entry/ownership problem, not just an exit problem.

Structural fix: use the same index-runner state as a side-ownership guard. If an active PE runner exists, CE entries require confirmed index reversal, not just CE premium recovery. If an opposite position is already open, the system should either close/handoff after confirmed runner flip or block new opposite materialization until the index runner is invalidated.

### 3. LATE_ENTRY: the engine waits for recovery proof after the runner has matured

| Date | Leg | Class | Side | Time | Pts | Entry | Delay min | Coverage | Entry tag |
|---|---|---|---|---|---|---|---|---|---|
| 2026-04-16 | 4 | Major Runner | PE | 11:08-12:07 | -173.7 | 11:46:16 | 38.3 | 0.35 | TR_STABLE_RETRY_BASKET |
| 2026-04-20 | 3 | Major Runner | CE | 09:48-10:40 | 164.8 | 10:09:06 | 21.1 | 0.47 | TR_STABLE_RETRY_BASKET |
| 2026-04-24 | 1 | Runner | PE | 09:15-09:57 | -193.8 | 09:30:03 | 15.1 | 0.43 | MOMENTUM_RIDE_PE |
| 2026-05-05 | 13 | Runner | CE | 12:32-13:11 | 128.6 | 12:59:50 | 27.8 | 0.29 | TR_STABLE_RETRY_R |
| 2026-05-11 | 10 | Runner | CE | 13:06-13:54 | 85.5 | 13:45:57 | 40.0 | 0.17 | TR_STABLE_RETRY_BASKET |
| 2026-05-13 | 8 | Major Runner | CE | 10:59-11:45 | 126.2 | 11:42:22 | 43.4 | 0.06 | TR_STABLE_RETRY_BASKET |
| 2026-05-18 | 4 | Major Runner | CE | 10:47-11:39 | 166.7 | 11:38:31 | 51.5 | 0.01 | TR_STABLE_RETRY_R |
| 2026-05-21 | 7 | Major Runner | PE | 11:28-12:41 | -137.5 | 12:27:02 | 59.0 | 0.19 | TR_STABLE_RETRY_R |
| 2026-05-29 | 11 | Runner | PE | 11:54-12:16 | -88.0 | 12:15:21 | 21.4 | 0.03 | TR_STABLE_RETRY_BASKET |

Diagnosis: the entry layer is still organized around DVR recovery, STABLE_RETRY materialization, and candidate reservation. Those are good when the option is recovering from discount, but they detect some session runners only after the move is already mature. That creates late entries with low benchmark-window coverage.

Structural fix: add a side-aligned runner participation route. It should be allowed only when the index runner state is active and multi-strike same-side option confirmation exists. This prevents chasing random premium blasts while letting the engine join persistent NIFTY runner legs before DVR calls them exhausted.

### 4. MISSED: no route owns the runner leg

| Date | Leg | Class | Side | Time | Pts |
|---|---|---|---|---|---|
| 2026-04-17 | 11 | Runner | CE | 12:00-14:46 | 100.7 |
| 2026-04-23 | 3 | Runner | CE | 09:36-10:05 | 108.7 |
| 2026-04-23 | 4 | Major Runner | PE | 10:05-11:44 | -128.0 |
| 2026-04-23 | 7 | Runner | CE | 12:31-13:24 | 105.0 |
| 2026-04-27 | 9 | Runner | PE | 10:16-10:39 | -81.8 |
| 2026-04-27 | 13 | Runner | PE | 11:18-11:38 | -91.5 |
| 2026-04-28 | 8 | Runner | PE | 10:58-11:33 | -122.8 |
| 2026-04-28 | 10 | Runner | PE | 11:50-12:33 | -92.2 |
| 2026-05-05 | 6 | Major Runner | PE | 09:55-10:57 | -131.5 |
| 2026-05-07 | 2 | Runner | CE | 09:22-09:43 | 95.1 |
| 2026-05-11 | 6 | Runner | CE | 10:19-11:45 | 102.3 |
| 2026-05-12 | 3 | Runner | PE | 09:19-10:05 | -116.8 |
| 2026-05-15 | 3 | Runner | CE | 09:36-09:57 | 93.9 |
| 2026-05-20 | 1 | Runner | CE | 09:15-09:39 | 113.8 |
| 2026-05-20 | 7 | Major Runner | CE | 10:49-11:50 | 133.9 |
| 2026-05-21 | 1 | Runner | PE | 09:15-09:35 | -104.5 |
| 2026-05-21 | 5 | Runner | PE | 10:46-11:17 | -130.0 |
| 2026-05-26 | 1 | Runner | CE | 09:15-09:40 | 100.6 |
| 2026-05-27 | 5 | Runner | CE | 10:01-10:37 | 96.0 |

Diagnosis: missed runner legs are not all “bad filters.” Some are a missing state-machine path: no open position, index leg is moving, but neither DVR nor STABLE_RETRY materializes a trade because their predicates are recovery/scalp oriented.

Structural fix: the runner participation route should own these cases. It must be symmetric CE/PE, use NIFTY structure plus option basket confirmation, and be measured by the benchmark gate after every change.

## Proposed Structural Architecture

### A. Add a shared `IndexRunnerState`

The state should be computed from live/backtest NIFTY ticks using only information available at that timestamp. It must not read benchmark labels. Minimum fields:

| Field | Purpose |
|---|---|
| `side` | `CE` for bullish NIFTY runner, `PE` for bearish NIFTY runner |
| `state` | `none`, `candidate`, `active`, `digesting`, `invalidated` |
| `start_ts` | first structural break/continuation timestamp |
| `anchor_price` | runner start price |
| `extreme_price` | current high/low reached by the runner |
| `points` | absolute NIFTY movement from anchor |
| `duration_sec` | age of the runner |
| `giveback_points` / `giveback_ratio` | pullback from extreme |
| `slope_score` | short/medium rate-of-change confirmation |
| `reversal_score` | evidence that opposite side has taken control |

Initial structural thresholds should be broad and class-based, not day-specific: for example, active runner after sustained 45-80 NIFTY points with persistence, VWAP/EMA continuation, and controlled giveback; major runner after 100+ points or sustained duration. These values should be validated across the benchmark gate, not tuned to one date.

### B. Entry integration

1. Existing DVR/STABLE_RETRY remains the primary route.
2. If no position exists and `IndexRunnerState.active` exists, allow `RUNNER_PARTICIPATION` only when same-side option basket confirms sponsorship.
3. If the selected token is already too extended, do not force a direct buy; reserve the side and require a controlled digestion/reclaim entry.
4. Same-side reentry after a profitable exit is allowed only when a fresh post-exit index runner state is active and same-side basket confirms it. This protects 21-Apr style chop while fixing 23-Jun style second-runner misses.

### C. Exit integration

1. If held side matches `IndexRunnerState.active`, ladder/target exits should be deferred unless runner invalidation is proven.
2. `EXIT_LADDER_LOCK` should not fire only because option profit retraced while NIFTY runner remains structurally alive.
3. Release conditions should stay strict: opposite index runner, opposite basket sponsorship, held-token VWAP failure, hard loss, EOD, or bounded giveback after mature peak.
4. This should address 04-29, 05-13, 05-26 early exits without removing profit protection from chop days.

### D. Wrong-side prevention

1. If a PE runner is active, CE entries need confirmed index reversal before materialization.
2. If a CE runner is active, PE entries need confirmed index reversal before materialization.
3. Existing local option recovery evidence can contribute to reversal score, but cannot alone override the index-runner owner.

## Protected Regression Days

These days should be pinned in every local regression before a fix is promoted:

| Day | Why protected |
|---|---|
| 2026-04-16 | Valid PE runner and later CE runner; protects against late/over-held exits. |
| 2026-04-17 | CE runner coverage and mid-day CE runner miss; protects symmetric CE runner entry. |
| 2026-04-20 | Whipsaw with CE/PE/CE transitions; protects reversal ownership logic. |
| 2026-04-21 | Existing comments mention same-side profitable reentry guard; protects chop regression. |
| 2026-04-29 | CE major runner early exit; primary exit-hold case. |
| 2026-04-30 | PE runner wrong-side overlap and CE major runner hold; protects both side ownership and long runner hold. |
| 2026-05-04 | Whipsaw day with multiple real PE/CE runners; protects against one-side overbias. |
| 2026-05-12 | PE major runner wrong-side CE entry; primary side-ownership case. |
| 2026-05-13 | CE major runner early exit and later late entry; protects exit and reentry. |
| 2026-05-14 | CE major runner data-quality day; protects broad basket runner behavior. |
| 2026-05-18 | CE major runner late entry with adjacent wrong-side overlap; protects reversal timing. |
| 2026-05-21 | PE major runner late entry and wrong-side impulse; protects PE symmetry. |
| 2026-05-26 | PE major runner extreme early exit; primary long-duration hold case. |
| 2026-06-23 | Same-side reentry after profitable early PE; protects post-exit fresh-runner reentry. |
| 2026-06-24 | Multiple CE runners; protects mid-day runner participation. |
| 2026-06-25 | CE-to-PE runner transition; protects reversal handoff. |
| 2026-06-29 | Large PE runner; protects DVR exhausted-to-runner handoff. |

## Implementation Plan

1. Build `IndexRunnerState` as a small, deterministic module with unit tests on synthetic NIFTY tick paths.
2. Add a debug-only/export mode first so the state is visible in `fusion_events` without changing trades.
3. Wire exit veto for `EARLY_EXIT` cases first because it is easier to validate: same entry, better hold/release.
4. Wire wrong-side materialization block next: block opposite entries only while index runner is active and no reversal proof exists.
5. Wire runner participation entry last: this is the riskiest because it creates new entries, so it needs stricter basket confirmation and the widest regression sweep.
6. For each phase, compare local coverage report against this remote baseline and require fewer failures without increasing protected-day regressions.

## Current Conclusion

The system is not broken by one threshold. It lacks a shared runner lifecycle. DVR, STABLE_RETRY, and the order exit layer are each making locally reasonable decisions, but they do not share one answer to: “Is the NIFTY Runner/Major Runner still alive, and who owns it?”

That is the concrete gap to fix.
