# Runner Coverage Consolidation Plan - 2026-07-05

This is the working plan after the strategy-gate workshop. The goal is to stop fixing one date at a time and instead make Impulse Runner, Runner, and Major Runner capture a measured regression contract.

## Objective

Build a maintainable runner-aware trading loop that:

1. Captures structurally tradable NIFTY `Impulse Runner`, `Runner`, and `Major Runner` legs.
2. Does not optimize for scalp, sideways, or noise legs. Those can be traded only if they naturally pass risk filters, but they are not the design target.
3. Avoids premature ladder/target exits while the same-side index runner is still alive.
4. Blocks wrong-side entries inside active index runners unless reversal is proven.
5. Enforces one coherent open-position lifecycle per user: a second entry must be a valid close/reversal/handoff, not an accidental overlap.
6. Keeps DVR, STABLE_RETRY, and MOMENTUM_RIDE understandable as families, not a growing list of tag suffixes.
7. Proves every change with before/after runner coverage and protected regression days.

## Explicit Problem Statement

The current system has three structural problems:

| Problem | What it means | Required fix direction |
|---|---|---|
| Mismanaged multiple entry/exit gates | DVR, STABLE_RETRY, MOMENTUM_RIDE, basket-runner, direct-runner, and handoff variants all influence entry/exit differently through raw tag strings. | Consolidate into canonical families and lifecycles, then route behavior from taxonomy instead of ad-hoc tag suffixes. |
| Missed meaningful NIFTY legs | The system misses or enters late on documented `Impulse Runner`, `Runner`, and `Major Runner` legs, while scalp/sideways behavior is not the priority. | Add measured runner coverage as the regression contract and create a runner participation path only for live-observable structural runners. |
| Premature runner exits | Ladder or target exits close positions early even when the same-side NIFTY runner remains alive. | Add index-runner lifecycle awareness to exits so ladder exits defer until runner invalidation or bounded giveback/reversal evidence appears. |

## Current State

| Area | Status |
|---|---|
| Benchmark coverage gate | Implemented in `tools/benchmark_runner_gate.py`. |
| Remote baseline export | Available under `outputs/runner-workshop/remote-current/`. |
| Gate taxonomy parser | Implemented in `entry_tag_taxonomy.py`; no behavior change. |
| Runner-state prototype | Implemented in `index_runner_state.py`; Phase 1 exit-side integration exists locally. |
| Local replay reliability | Not yet trusted; latest focused replay returned zero orders and needs wiring/profile investigation. |
| Remote 173 | Treat as baseline/prod-like environment. Do not touch unless explicitly asked. |

## Ground Rules

1. Use one canonical local worktree: `/Users/<user>/Documents/Codex/2026-04-24/trade-engine-redesign/zatamap-loss-trade-guards-0701`.
2. No day-specific threshold hacks.
3. Scalp/sideways performance is secondary. Do not add rules just to increase scalp count.
4. Every behavior change must name the failure class it addresses: `EARLY_EXIT`, `LATE_ENTRY`, `MISSED`, `WRONG_SIDE_OVERLAP`, `DUPLICATE_OVERLAP`, or `LOW_WINDOW_CAPTURE`.
5. Every behavior change must include a code comment with the structural reason and representative RCA dates.
6. Any fix touching a commented historical guard must regression-test the day named in that comment.
7. Remote 173 is used for baseline comparison unless the user explicitly asks to deploy or run there.

## Phase 0 - Fix The Local Replay Lab

Reason: local focused replay returned zero orders for days where remote baseline has trades. We cannot trust local impact analysis until this is fixed.

Tasks:

| Task | Expected output |
|---|---|
| Inspect `.trade-api-runs/index-runner-phase1-20260705/*/app.log` and `replay.log`. | Identify why local replay generated zero orders. |
| Check profile/user cloning, enabled strategy flags, DB connection, and instrument/tick access. | Confirm same input path as remote backtest. |
| Run one known trade day, for example 2026-04-29, and compare order count against remote baseline. | Local replay produces expected non-zero trades. |
| Only after local replay is healthy, restart local before/after impact loops. | Local results become usable evidence. |

Exit criteria:

- At least one known remote-trading day produces comparable local trades.
- Local run logs show strategy evaluation and order materialization, not only tick replay.

## Phase 1 - Normalize Strategy Reporting

Reason: the dashboard currently splits raw tags into many pseudo-strategies.

Tasks:

| Task | Expected output |
|---|---|
| Use `entry_tag_taxonomy.py` to group historical trades by family and lifecycle. | Leaderboard by `family:lifecycle`, not raw tag. |
| Compare raw leaderboard vs normalized leaderboard. | Identify weak lifecycle groups such as `DVR_RECOVERY:recovery`. |
| Add a report/export utility if needed. | Repeatable strategy taxonomy report. |

Exit criteria:

- We can answer whether a problem is with `DVR_RECOVERY` overall or only plain DVR recovery.
- No trading behavior changes in this phase.

## Phase 2 - Make Runner Coverage The Regression Gate

Reason: profitability alone hides missed Impulse Runner, Runner, and Major Runner legs and early exits.

Tasks:

| Task | Expected output |
|---|---|
| Run benchmark coverage on the remote baseline export. | Current failure table by day and class. |
| Run benchmark coverage on local candidate runs. | Before/after comparison. |
| Track status counts: `CAPTURED`, `LATE_ENTRY`, `EARLY_EXIT`, `MISSED`, `WRONG_SIDE_OVERLAP`, `DUPLICATE_OVERLAP`. | Objective improvement or regression signal. |

Exit criteria:

- Every fix is evaluated by runner coverage, not only P&L.
- Candidate result cannot be promoted if it improves one day but increases critical misses elsewhere.

## Phase 3 - Structural Fix Class 1: Early Exit

Failure class: `EARLY_EXIT`.

Representative cases:

| Date | Leg | Issue |
|---|---|---|
| 2026-04-29 | CE Major Runner 09:23-10:28 | `EXIT_LADDER_LOCK` closed while index runner continued. |
| 2026-05-13 | CE Major Runner 09:40-10:34 | DVR plain recovery exited after a few minutes. |
| 2026-05-26 | PE Major Runner 11:39-14:54 | DVR/STABLE_RETRY handoff exited far before runner end. |

Design:

- Use `IndexRunnerState` to detect whether the held side still matches an active NIFTY runner.
- Veto only ladder/target-style exits, not hard stop-loss, EOD, or proven reversal exits.
- Release the hold if index runner invalidates, opposite basket sponsorship appears, held-token VWAP fails, or bounded giveback becomes unacceptable.

Exit criteria:

- Early-exit count decreases on the benchmark gate.
- 04-29, 05-13, and 05-26 improve without worsening whipsaw protection days.

## Phase 4 - Structural Fix Class 2: Position Ownership, Wrong-Side Overlap, And Duplicate Entry

Failure classes: `WRONG_SIDE_OVERLAP`, `DUPLICATE_OVERLAP`.

Representative cases:

| Date | Needed runner | Problem |
|---|---|---|
| 2026-05-12 | PE Major Runner | CE trade opened inside active PE runner. |
| 2026-05-11 | PE Major Runner | CE position survived into PE runner. |
| 2026-04-30 | PE Runner | CE overlap occurred during bearish runner. |

Design:

- If `IndexRunnerState` is active for PE, CE entries require confirmed index reversal.
- If `IndexRunnerState` is active for CE, PE entries require confirmed index reversal.
- Opposite option recovery alone is not enough; it must prove index ownership transfer.
- A user can have only one open strategy lifecycle. If an open position exists, a new signal must become one of: ignore, reinforce existing side without a new order, close existing position, or confirmed reversal handoff.
- `orders_tracker`, order cache, and strategy open-position state must agree before any new entry is allowed.

Exit criteria:

- Wrong-side overlap count decreases.
- Duplicate/open-position overlap count is zero.
- Reversal days like 2026-04-20 and 2026-05-04 do not lose valid flips.

## Phase 5 - Structural Fix Class 3: Late Entry / Missed Impulse And Runner Legs

Failure classes: `LATE_ENTRY`, `MISSED`.

Representative cases:

| Date | Runner | Issue |
|---|---|---|
| 2026-04-16 | PE Major Runner 11:08-12:07 | Entry came too late. |
| 2026-04-23 | PE Major Runner 10:05-11:44 | No route owned the runner. |
| 2026-06-23 | PE Major Runner 11:20-12:22 | Same-side profit reentry guard blocked fresh runner. |
| 2026-06-24 | CE runners | Need timely CE runner participation. |
| 2026-06-29 | PE Runner 11:04-12:41 | Entry was too late / runner was treated as exhausted. |

Design:

- Add a runner participation route only when live-observable index runner state and same-side multi-strike option confirmation agree.
- If selected token is already too extended, reserve the side and wait for controlled digestion/reclaim instead of chasing.
- Same-side reentry after profit is allowed only when a fresh post-exit runner proves itself.
- Treat `Impulse Runner` as a first-class target when the move is persistent and option basket confirmation exists. Do not require the system to wait until an impulse has already matured into a late recovery.

Exit criteria:

- `MISSED` and `LATE_ENTRY` decrease without increasing bad trades on protected chop days.
- 21-Apr same-side reentry guard remains protected.

## Protected Regression Days

These days must be included in every serious candidate run:

| Day | Why |
|---|---|
| 2026-04-16 | Valid PE and CE runner behavior. |
| 2026-04-17 | CE runner coverage and mid-day runner miss. |
| 2026-04-20 | Whipsaw with real reversals. |
| 2026-04-21 | Same-side reentry guard protection. |
| 2026-04-23 | Multiple missed runners. |
| 2026-04-24 | Opening PE runner and wrong-side risk. |
| 2026-04-29 | CE major runner early exit. |
| 2026-04-30 | PE runner and CE major runner. |
| 2026-05-04 | High-opportunity whipsaw. |
| 2026-05-12 | PE major runner, wrong-side CE problem. |
| 2026-05-13 | CE major runner early exit. |
| 2026-05-14 | Data quality day but important major runner context. |
| 2026-05-18 | CE/PE runner transitions. |
| 2026-05-21 | Opening and mid-day PE runners. |
| 2026-05-26 | Long PE major runner early exit. |
| 2026-06-23 | Same-side reentry after profit and late-day runner. |
| 2026-06-24 | CE runner participation. |
| 2026-06-25 | CE/PE runner transitions. |
| 2026-06-29 | PE runner missed/late. |

## Promotion Criteria

A candidate fix is promotable only if:

1. Local replay lab is trusted.
2. Unit tests and py-compile pass.
3. Impulse Runner, Runner, and Major Runner coverage improves for the target failure class.
4. Protected regression days do not show new critical failures.
5. Strategy taxonomy report does not show a new raw-tag variant created unnecessarily.
6. Duplicate open-position overlap remains zero.
7. Paper/live mode separation remains intact.
8. Stop-loss, EOD, paper DB-only behavior, and broker isolation are verified separately before any paper/live deploy.

## Immediate Next Step

Do Phase 0 first. The next coding move should not be another entry threshold change; it should be making local replay trustworthy again. Once local replay produces comparable trades for known baseline days, run the taxonomy report and benchmark gate before making the next behavior change.
